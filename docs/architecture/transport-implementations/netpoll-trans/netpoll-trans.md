# 基于 netpoll 的传输实现（netpoll-trans）

> 本文是 `transport-implementations` 域下的叶子子系统文档。域级总览见 `../transport-implementations.md`。
> 本文只展开"netpoll 事件驱动服务端、连接扩展、拨号、netpoll 缓冲适配"，不展开通用收发编排（见 `../remote-transport/trans-handler-pipeline/trans-handler-pipeline.md`）与编解码（见 `../remote-transport/codec-payload/codec-payload.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 服务端 handler 工厂 | `NewSvrTransHandlerFactory`：基于默认 handler + netpoll 连接扩展 | `pkg/remote/trans/netpoll/server_handler.go:27` |
| 客户端 handler 工厂 | `NewCliTransHandlerFactory`：基于默认 handler + netpoll 连接扩展 | `pkg/remote/trans/netpoll/client_handler.go:27` |
| netpoll 拨号器 | `NewDialer` 返回 `netpoll.NewDialer()` | `pkg/remote/trans/netpoll/dialer.go:26` |
| 连接扩展 | `netpollConnExtension` 实现 `trans.Extension`：读写超时、缓冲创建/释放、超时与远端关闭错误判定 | `pkg/remote/trans/netpoll/conn_extension.go:37` |
| 事件驱动服务端 | `transServer`：netpoll EventLoop 驱动，OnPrepare=建连、onConnRead=读循环 | `pkg/remote/trans/netpoll/trans_server.go:54` |
| 优雅关闭 | Shutdown：关 listener → 通知活跃连接优雅关闭 → 等待 100ms → 停 EventLoop | `pkg/remote/trans/netpoll/trans_server.go:107` |
| 连接计数 | `connCount utils.AtomicInt` 随建连/断连增减 | `pkg/remote/trans/netpoll/trans_server.go:61/155/183` |
| netpoll 缓冲适配 | 把 `netpoll.Reader/Writer` 包成 `remote.ByteBuffer`（零拷贝链表缓冲） | `pkg/remote/trans/netpoll/bytebuf.go` |
| panic 兜底 | `transRecover`：日志记录，读循环 panic 可选择向上传播 | `pkg/remote/trans/netpoll/trans_server.go:191` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `svrTransHandlerFactory` / `cliTransHandlerFactory` | `server_handler.go:24` / `client_handler.go:24` | 工厂；内部委托 `trans.NewDefaultSvr/CliTransHandler(opt, NewNetpollConnExtension())` |
| `netpollConnExtension` | `conn_extension.go:37` | 实现 `trans.Extension`，把 netpoll 连接能力适配到通用 handler |
| `transServer` | `trans_server.go:54` | netpoll 传输服务端；持有 `evl netpoll.EventLoop`、`ln net.Listener`、`connCount` |
| `netpollTransServerFactory` | `trans_server.go:43` | 实现 `remote.TransServerFactory` |
| `NewWriterByteBuffer` / `NewReaderByteBuffer` | `bytebuf.go` | 把 netpoll Writer/Reader 包装为 `remote.ByteBuffer` |

## 3. 关键调用链

### 3.1 服务端启动与事件循环（BootstrapServer）

1. `transServer.BootstrapServer`（`trans_server.go:78`）：构造 `netpoll.Option`（`WithIdleTimeout(MaxConnectionIdleTime)`、`WithReadTimeout(ReadWriteTimeout)`，`:84-85`）。
2. `netpoll.NewEventLoop(ts.onConnRead, WithOnPrepare(ts.onConnActive), ...)`（`:90`）——把读回调与建连回调注册进 netpoll EventLoop。
3. `netpoll.ConvertListener(ts.ln)`（`:97`）转换 listener，使关闭 listener 能停止 EventLoop。
4. `ts.evl.Serve(ts.ln)`（`:103`）进入事件循环。

### 3.2 连接建/读/断回调

- `onConnActive`（`trans_server.go:148`）：`conn.AddCloseCallback(onConnInactive)`（`:151`）注册断连回调 → `connCount.Inc`（`:155`）→ `transHdlr.OnActive`（`:156`）；失败 `onError` + `conn.Close`。
- `onConnRead`（`trans_server.go:165`）：`transHdlr.OnRead`（`:170`）；返回 error 则 `onError` 并 `conn.Close`（`:172-176`）。
- `onConnInactive`（`trans_server.go:181`）：`connCount.Dec`（`:183`）→ `transHdlr.OnInactive`。

### 3.3 连接扩展适配

- `SetReadTimeout`（`conn_extension.go:40`）：客户端用 `trans.GetReadTimeout(cfg)`，服务端用 `cfg.ReadWriteTimeout()`（`:43/45`）。
- `NewWriteByteBuffer`（`conn_extension.go:50`）/`NewReadByteBuffer`（`:55`）：把 `conn.(netpoll.Connection).Writer()/Reader()` 包成 ByteBuffer。
- `IsTimeoutErr`（`:68`）判 `netpoll.ErrReadTimeout`；`IsRemoteClosedErr`（`:73`）判 `netpoll.ErrConnClosed` 或 `syscall.EPIPE`。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `MaxConnectionIdleTime` | netpoll `WithIdleTimeout` 空闲连接回收 | `trans_server.go:84` |
| `ReadWriteTimeout` | netpoll `WithReadTimeout`；服务端 Read 超时 | `trans_server.go:85` |
| `ExitWaitTime` | Shutdown 的 `context.WithTimeout` 优雅等待上限 | `trans_server.go:111` |
| `ExitWaitTime` 内固定 sleep 100ms | 关 listener 后等待在途请求以减少对端 EOF | `trans_server.go:130` |

## 5. 错误与重试语义

- **OnRead 出错**：`onConnRead` 调 `onError` 后 `conn.Close`（`trans_server.go:172-176`），连接级失败不重试。
- **建连出错**：`onConnActive` 调 `onError` + `conn.Close`（`:157-159`）。
- **panic 兜底**：`transRecover`（`:191`）recover 并打日志；`onConnRead` 的 panic `propagatePanic=true` 会重新 `panic`（框架层 bug 让其 crash），建/断连回调为 false。
- **超时/远端关闭判定**：由 `netpollConnExtension.IsTimeoutErr/IsRemoteClosedErr` 识别，供上层包装为 `ErrRPCTimeout`/打 `RemoteClosedTag`。

## 6. 并发细节

- **goroutine 边界**：netpoll EventLoop 自身持有 reactor/goroutine（外部库）；本层不直接起业务 goroutine，读回调 `onConnRead` 在 netpoll 事件 goroutine 内同步执行 `OnRead`。
- **连接计数**：`connCount utils.AtomicInt` 原子增减（`trans_server.go:155/183`），无锁。
- **优雅关闭**：`Shutdown` 用 `sync.Mutex` 保护（`trans_server.go:88/108`），先关 listener 阻止新连接，再通知活跃连接，最后停 EventLoop——避免在途请求被立即切断。
- **context 传播**：建连用 `context.Background()`（`:149`），RPC 超时经 `SetReadTimeout` 落到 netpoll 连接读超时。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/trans/netpoll/` 下 server/client handler 工厂、连接扩展、拨号、transServer、bytebuf 适配。

**Out-of-Scope（不在本仓库源码内）**
- netpoll 事件循环、reactor、`netpoll.Connection`/`Reader`/`Writer`/`EventLoop`/`ConvertListener` 的实现：外部库 `github.com/cloudwego/netpoll`。
- 通用收发编排与编解码：分别见 trans-handler-pipeline、codec-payload 叶子。

## 8. 与相邻子系统交互

- **上游**：`internal/server|client/option.go` 选 netpoll 时用 `NewSvr/CliTransHandlerFactory` 与 `NewDialer`；`remotesvr.Server` 用 `NewTransServerFactory`。
- **下游（外部 netpoll）**：`transServer` 注册 OnPrepare/onConnRead 回调给 netpoll；`netpollConnExtension` 把 netpoll Connection 能力适配进通用 handler。
- **横向**：复用 `trans.NewDefaultSvr/CliTransHandler` 与 `trans.Extension` 接口（与 gonet 共享），仅扩展实现不同。
- **方向**：netpoll 事件 → onConnRead → transHdlr.OnRead（通用编排）→ 编解码 → netpoll Writer 写回。

## 9. 语言专项适配口径（Go）

- **并发模型**：基于外部 netpoll 的 reactor 模式（epoll 边缘触发、工作者协程），本层只注册回调，不管理 goroutine；这是与 gonet"每连接一 goroutine"并列的另一种并发模型。
- **适配器模式**：`netpollConnExtension` 把 netpoll 特有能力（连接读超时、Reader/Writer、错误类型）适配为 `trans.Extension` 通用接口，使通用 handler 不感知 netpoll/gonet 差异——依赖倒置。
- **internal 边界**：`pkg/remote/trans`（含 `default_*_handler.go`、`Extension`）为共享层；`netpoll/`、`gonet/` 为各自传输实现，平级并列。
- **panic 隔离**：读循环 panic 重新上抛（框架 bug fail-fast），建/断连回调只记录——区分"框架自身错误"与"连接级错误"的处置。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| netpoll 事件驱动传输架构图 | `netpoll-trans-architecture.html` | architecture | standard |
| 建连/读/断回调时序图 | `netpoll-trans-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 子目录。事件回调时序清晰故补 sequence；无显式业务状态机，不单列 lifecycle。
