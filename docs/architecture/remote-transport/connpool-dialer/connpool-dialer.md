# 连接池、拨号与客户端/服务端远程会话（connpool-dialer）

> 本文是 `remote-transport` 域下的叶子子系统文档。域级总览见 `../remote-transport.md`。
> 本文只展开"连接池抽象与实现、拨号抽象、remotecli 客户端会话、remotesvr 服务端会话"，不展开收发 handler 编排（见 `../trans-handler-pipeline/trans-handler-pipeline.md`）与具体传输实现（见 `transport-implementations` 域）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| ConnPool 抽象 | Get/Put/Discard/Close 连接池接口；ConnOption 携带 Dialer 与 ConnectTimeout | `pkg/remote/connpool.go:32/26` |
| LongConnPool 抽象 | 在 ConnPool 上增加 `Clean(network, address)` 按地址清理 | `pkg/remote/connpool.go:48` |
| 连接池观察接口 | ConnPoolReporter/RawConn/IsActive 扩展能力 | `pkg/remote/connpool.go:56/61/66` |
| Dialer 抽象 | `DialTimeout(network, address, timeout)`；默认 `net.DialTimeout` 实现 | `pkg/remote/dialer.go:25/30` |
| SynthesizedDialer | 用一个 DialFunc 合成 Dialer，便于注入自定义拨号 | `pkg/remote/dialer.go:37` |
| 长连接池实现 | FILO 栈取连接、FIFO 队列驱逐过期连接，按 peer 维度分池 | `pkg/remote/connpool/long_pool.go:119` |
| 短连接池实现 | 每次 Get 新建连接，release 即关 | `pkg/remote/connpool/short_pool.go:53` |
| 空连接池/Reporter | DummyPool/DummyReporter 占位实现 | `pkg/remote/connpool/dummy.go` |
| remotecli 客户端会话 | Client 接口（Send/Recv/Recycle），绑连接与 handler，对象池复用 | `pkg/remote/remotecli/client.go:32/45` |
| 连接包装器 | ConnWrapper 封装"取连接/归还连接"策略，按错误与标签决定 Put/Discard/Close | `pkg/remote/remotecli/conn_wrapper.go:45` |
| 流会话 | NewStream 按 GRPC/TTHeaderStreaming 分流创建客户端流；StreamConnManager | `pkg/remote/remotecli/stream.go:32/75` |
| remotesvr 服务端会话 | Server 接口（Start/Stop/Address），建 listener 并 BootstrapServer | `pkg/remote/remotesvr/server.go:30/36` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `remote.ConnPool` | `connpool.go:32` | 连接池根接口；`Put` 注释明确"conn 的 Close 可能已被调用"，需幂等 |
| `remote.Dialer` | `dialer.go:25` | 拨号根接口；`NewDefaultDialer` 用标准库 `net.DialTimeout` |
| `pool`（长连接池内部） | `long_pool.go:119` | 维护 `idleList`（FILO Get/Put、FIFO Evict），`sync.RWMutex`，`maxIdle`/`maxIdleTimeout` |
| `longConn` | `long_pool.go:72` | 包装 net.Conn，实现 `RawConn`/`IsActive`/`Expired`，带 `deadline` |
| `ShortPool` | `short_pool.go:53` | 短连接池；Get 拨号、Put/Close 即释放 |
| `remotecli.Client`/`client` | `client.go:32/45` | 一次 RPC 的客户端会话；`clientPool` sync.Pool 复用 |
| `ConnWrapper` | `conn_wrapper.go:45` | 持有 connPool 与当前 conn；`ReleaseConn` 决策归还/丢弃/关闭 |
| `StreamConnManager` | `stream.go:75` | 嵌入 `ConnReleaser`，管理流底层连接生命周期 |
| `remotesvr.server` | `server.go:36` | 服务端会话；`Start` 用 `gofunc.GoFunc` 起 goroutine 跑 BootstrapServer |

## 3. 关键调用链

### 3.1 客户端建连与会话（NewClient）

1. `remotecli.NewClient`（`client.go:52`）：`NewConnWrapper(opt.ConnPool)` 包装连接池 → 逐个执行 `StreamingMetaHandlers.OnConnectStream`（`:56`）→ `cm.GetConn`（`:62`）取连接。
2. `ConnWrapper.GetConn`（`conn_wrapper.go:58`）：有连接池走 `getConnWithPool`（`:113`）→ `cp.Get(ctx, network, address, ConnOption{Dialer, ConnectTimeout})`（`:122`），并记录 `stats.ClientConnStart/Finish`；无连接池走 `getConnWithDialer`（`:131`）→ `d.DialTimeout`（`:140`）。
3. 成功后 `clientPool.Get`（`client.go:69`）复用一个 `client`，`init` 绑定 handler/ConnWrapper/conn。

### 3.2 长连接池取还与驱逐

- `pool.Get`（`long_pool.go:129`）：加锁后从栈顶（FILO）向下找第一个 `IsActive()` 的空闲连接返回，遇到失效的直接 `Close`；找不到则新建。
- `pool.Put`（`long_pool.go:156`）：未超 `maxIdle` 则打 `deadline = now+maxIdleTimeout` 压回栈顶。
- `pool.Evict`（`long_pool.go:170`）：从队列头（FIFO）起关闭 `Expired()` 的连接，保留 `minIdle` 个；由共享 ticker 周期触发。

### 3.3 请求后连接回收策略（ReleaseConn）

`ConnWrapper.ReleaseConn`（`conn_wrapper.go:81`）：
- `err == nil` 且连接未打 `ConnResetTag` 且非 Oneway → `connPool.Put`（`:91`）复用；
- 出错、ConnResetTag、Oneway → `connPool.Discard`（`:89/94`）丢弃；
- 无连接池 → `conn.Close()`（`:97`）。
- 最后 `zero()` 并把 ConnWrapper 放回 `connWrapperPool`。

### 3.4 服务端启动

`remotesvr.server.Start`（`server.go:54`）→ `buildListener`（`:70`，优先用 `opt.Listener`，否则 `transSvr.CreateListener`）→ `gofunc.GoFunc`（`:66`）起 goroutine 执行 `transSvr.BootstrapServer(ln)`，错误经带缓冲 chan 返回。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `ConnOption.ConnectTimeout` | 来自 `ri.Config().ConnectTimeout()`，拨号超时 | `conn_wrapper.go:60` |
| `pool.maxIdle` / `minIdle` | 长连接池空闲上下限；超 maxIdle 不再回收 | `long_pool.go:123/124` |
| `pool.maxIdleTimeout` | 空闲超过即被 Evict 关闭 | `long_pool.go:125` |
| `opt.GRPCStreamingConnPool` | gRPC 流专用连接池，缺省回退 `opt.ConnPool` | `stream.go:45` |
| `opt.Listener` | 服务端可外部传入 listener（如 systemd socket） | `server.go:71` |

## 5. 错误与重试语义

- **取连接失败**：`getConnWithPool/getConnWithDialer` 失败统一包装为 `kerrors.ErrGetConnection.WithCause(err)`（`conn_wrapper.go:125/143`）；`NewClient` 对已是 `ErrGetConnection` 的错误原样返回（`client.go:64`）。
- **无目的地址**：`ri.To().Address()==nil` 返回 `ErrNoDestAddress`（`conn_wrapper.go:117/136`）。
- **失效连接**：长连接池 `Get` 遇到 `!IsActive()` 直接 `Close` 并继续找（`long_pool.go:144`），对调用方透明。
- **归还幂等**：`Put` 注释明确"conn 的 Close 可能已被调用"，实现需容忍重复 Close。
- **重试**：本层不负责 RPC 重试；连接失败上抛由 client 侧 retry 中间件决策。

## 6. 并发细节

- **goroutine 边界**：`remotesvr.Start` 用 `gofunc.GoFunc` 起一个 BootstrapServer goroutine（`server.go:66`），由 `Stop` 关闭；连接级读 goroutine 归传输实现持有。
- **连接池锁**：`pool` 用 `sync.RWMutex`（`long_pool.go:121`）；写路径 Get/Put/Evict/Close 用 `Lock`，只读 Len/Dump 用 `RLock`。`idleList` 是切片栈，操作在锁内完成。
- **对象池**：`client`（`client.go:39`）、`ConnWrapper`（`conn_wrapper.go:31`）均 sync.Pool 复用，`Recycle`/`ReleaseConn` 末尾 `zero()` 清空字段。
- **过期清理**：`Evict` 由 `getSharedTicker`（`long_pool.go:46`）周期触发，按 FIFO 关闭过期连接，与 Get/Put 的 FILO 分离，避免热点连接被过早回收。
- **context 传播**：`Get(ctx, ...)` 与拨号超时透传；RPC 超时不在本层。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/connpool.go`、`dialer.go` 抽象；`pkg/remote/connpool/{long_pool,short_pool,dummy,reporter,utils}.go` 实现。
- `pkg/remote/remotecli/{client,conn_wrapper,stream}.go` 客户端会话；`pkg/remote/remotesvr/server.go` 服务端会话。

**Out-of-Scope（不在本仓库源码内）**
- netpoll 连接与 eventloop、mux 连接池（`pkg/remote/trans/netpollmux/mux_pool.go`、`nphttp2/conn_pool.go`）见 `transport-implementations` 域对应叶子。
- 真正的 `net.Conn` 系统调用由标准库 `net`/外部 netpoll 提供。
- 客户端重试/熔断/负载均衡选址由 `client` 包与 `pkg/{retry,loadbalance}` 负责，本层只消费 `ri.To().Address()`。

## 8. 与相邻子系统交互

- **上游（client 包）**：`client/client.go` 用 `remotecli.NewClient` 建会话，调用 `Send`/`Recv`；连接池实例由 `internal/client/option.go` 依据配置构造注入 `opt.ConnPool`。
- **下游（trans-handler-pipeline）**：`client.transHdlr` 即装配好的 `TransPipeline`，Send/Recv 委托其 Write/Read。
- **下游（传输实现）**：`ConnWrapper.GetConn` 经 `opt.Dialer` 或 `ConnPool.Get` 拿到裸 `net.Conn`，再由 handler 的 `ext.NewRead/WriteByteBuffer` 绑定成缓冲。
- **方向**：client 调用 → NewClient 取连接 → Send(Write) → Recv(Read) → ReleaseConn 归还/丢弃。

## 9. 语言专项适配口径（Go）

- **并发模型**：连接池是经典"对象池 + 互斥锁"并发模型，非 K8s Reconcile；`idleList` 用切片而非 channel，锁粒度为单 peer 池。FILO 取连接使热点连接保持热，FIFO 驱逐防止冷连接长期占内存——这是高性能网络框架的典型取舍。
- **goroutine 边界**：服务端 BootstrapServer 单 goroutine 长驻，`gofunc.GoFunc` 统一 panic 兜底；连接读循环 goroutine 在传输实现层，本层不管理其生命周期。
- **对象池/sync.Pool**：`client`/`ConnWrapper` 会话对象池化降低分配；契约是用完必须 `zero()` 后 Put，禁止跨请求持有。
- **接口与依赖倒置**：`remote.ConnPool`/`remote.Dialer` 为框架侧抽象，具体长/短连接池在 `pkg/remote/connpool` 子包，由 option 注入；`RawConn`/`IsActive` 作为可选能力接口（type assertion 探测）。
- **internal 边界**：`pkg/remote/connpool` 为公开子包；`utils.SharedTicker`/`MaxCounter` 为其内部辅助。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 连接池/拨号/客户端会话架构图 | `connpool-dialer-architecture.html` | architecture | showcase |
| 连接取还与回收决策数据流图 | `connpool-dialer-dataflow.html` | dataflow | standard |
| 长连接状态生命周期图 | `connpool-dialer-lifecycle.html` | lifecycle | standard |

JSON IR 源文件位于 `json/` 子目录。dataflow 与 lifecycle 因多分支/跨 lane 连线在 showcase 档下布局校验不通过，按规范降为 standard 渲染成功；无跨方消息交互，不单列 sequence。
