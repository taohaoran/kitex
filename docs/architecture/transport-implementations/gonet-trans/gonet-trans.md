# 基于标准库 net 的传输实现（gonet-trans）

> 本文是 `transport-implementations` 域下的叶子子系统文档。域级总览见 `../transport-implementations.md`。
> 本文只展开"标准库 net 每连接一 goroutine 的服务端、连接扩展、拨号、缓冲适配"，不展开通用收发编排（见 `../remote-transport/trans-handler-pipeline/trans-handler-pipeline.md`）与编解码（见 `../remote-transport/codec-payload/codec-payload.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 服务端 handler 工厂 | `NewSvrTransHandlerFactory`：默认 handler + gonet 连接扩展 | `pkg/remote/trans/gonet/server_handler.go:27` |
| 客户端 handler 工厂 | `NewCliTransHandlerFactory`：默认 handler + gonet 连接扩展 | `pkg/remote/trans/gonet/client_handler.go:27` |
| gonet 拨号器 | 基于标准库 `net.Dialer` 的拨号 | `pkg/remote/trans/gonet/dialer.go` |
| 连接扩展 | `gonetConnExtension`：读写超时（客户端设 ReadDeadline）、缓冲创建、超时/远端关闭错误判定 | `pkg/remote/trans/gonet/conn_extension.go:37` |
| 每连接 goroutine 服务端 | `transServer.BootstrapServer`：Accept 循环 + `go serveConn` 每连接一 goroutine | `pkg/remote/trans/gonet/trans_server.go:75/93` |
| 手动读超时 | `refreshIdleDeadline`（默认 2 分钟空闲）与 `refreshReadDeadline`（业务 ReadWriteTimeout） | `pkg/remote/trans/gonet/trans_server.go:175/186` |
| 优雅关闭 | Shutdown：置 shutdown 标志 → 关 listener → 通知活跃连接 → ticker 轮询连接数归零 | `pkg/remote/trans/gonet/trans_server.go:133` |
| gonet 缓冲 | 把标准库 net.Conn Reader/Writer 包成 `remote.ByteBuffer` | `pkg/remote/trans/gonet/bytebuffer.go` |
| 服务端连接封装 | `svrConn`（`newSvrConn`）封装 conn 供读写 | `pkg/remote/trans/gonet/conn.go` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `gonetTransServerFactory` | `trans_server.go:44` | 实现 `remote.TransServerFactory` |
| `transServer` | `trans_server.go:55` | gonet 传输服务端；持有 `ln net.Listener`、`shutdown` 标志、`connCount` |
| `gonetConnExtension` | `conn_extension.go:37` | 实现 `trans.Extension`；客户端用 `net.Conn.SetReadDeadline` 控超时 |
| `newSvrConn` | `conn.go` | 服务端连接封装，提供 Reader/Writer |

## 3. 关键调用链

### 3.1 服务端 Accept 与每连接协程（BootstrapServer）

1. `transServer.BootstrapServer`（`trans_server.go:75`）：`for { ln.Accept() }`（`:84`）阻塞接受连接。
2. 接受成功后 `go ts.serveConn(context.Background(), conn)`（`:93`）——每连接一个 goroutine。
3. Accept 出错时若 `shutdown==1` 则正常返回（`:86-88`），否则报错退出。

### 3.2 连接处理协程（serveConn）

1. `serveConn`（`trans_server.go:97`）：`connCount.Inc`（`:100`）→ `newSvrConn`（`:101`）→ `transHdlr.OnActive`（`:110`）。
2. 进入 `for` 循环（`:115`）：`refreshIdleDeadline`（`:117`）设空闲读超时 → `bc.r.Peek(1)`（`:118`）阻塞等下一帧 → `refreshReadDeadline`（`:123`）按业务超时覆盖 → `transHdlr.OnRead`（`:125`）处理一请求。
3. defer 中：出错 `onError`（`:104`）、`connCount.Dec`（`:106`）、`bc.Close`（`:107`）。

### 3.3 连接扩展适配

- `SetReadTimeout`（`conn_extension.go:40`）：仅客户端设 `SetReadDeadline`（`:43`）；服务端在 `serveConn` 循环内自行设置。
- `NewReadByteBuffer`（`:60`）/`NewWriteByteBuffer`（`:55`）：把 `conn.(bufioxReadWriter)` 的 Reader/Writer 包成 ByteBuffer。
- `IsTimeoutErr`（`:73`）判 `net.Error.Timeout()`；`IsRemoteClosedErr`（`:83`）判 `io.EOF`/`EPIPE`/`net.ErrClosed`。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `MaxConnectionIdleTime` | 空闲读超时下限；小于 2 分钟时用 2 分钟兜底 | `trans_server.go:178-182` |
| `ReadWriteTimeout`（RPCConfig） | 每请求处理前覆盖读 deadline；0 表示不限制 | `trans_server.go:186-192` |
| `ExitWaitTime` | Shutdown 等待活跃连接关闭的 context 超时 | `trans_server.go:139` |
| `defaultShutdownTicker` | Shutdown 轮询连接数归零的 ticker 周期 | `trans_server.go:151` |

## 5. 错误与重试语义

- **OnRead/serveConn 出错**：直接 return 退出循环，defer 关闭连接（`trans_server.go:126-128/107`），连接级失败不重试。
- **Accept 出错**：shutdown 时静默返回，否则记录错误并退出 BootstrapServer（`:90-91`）。
- **panic 兜底**：`transRecover`（`:195`）recover 并打日志，不向上传播（与 netpoll 读循环传播 panic 不同）。
- **超时/远端关闭**：由 `gonetConnExtension.IsTimeoutErr/IsRemoteClosedErr` 识别。

## 6. 并发细节

- **goroutine 边界**：`BootstrapServer` 单 Accept goroutine + 每连接一个 `serveConn` goroutine（`trans_server.go:93`）；连接断开时该 goroutine 自然退出，无泄漏。高并发下 goroutine 数量等于并发连接数，这是与 netpoll reactor 模型的核心差异。
- **shutdown 标志**：`atomic.StoreUint32(&ts.shutdown, 1)`（`:134`），Accept goroutine 原子读判断是否退出。
- **优雅关闭**：Shutdown 用 `sync.Mutex`（`:136`），关 listener 后 ticker 周期轮询 `connCount.Value()==0`（`:157`）退出，或 `ctx.Done()` 超时。
- **读超时手动管理**：gonet 无事件库，用 `net.Conn.SetReadDeadline` 手动控制（`refreshIdleDeadline`/`refreshReadDeadline`），每请求前刷新。
- **context 传播**：每连接 `context.Background()`（`:93`）；RPC 超时落到 ReadDeadline。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/trans/gonet/` 下 server/client handler 工厂、连接扩展、拨号、transServer、bytebuffer、conn。

**Out-of-Scope（不在本仓库源码内）**
- 标准库 `net` 包的 socket/Accept/Read/Write 系统调用（Go 标准库）。
- 通用收发编排与编解码：见 trans-handler-pipeline、codec-payload 叶子。

## 8. 与相邻子系统交互

- **上游**：`internal/server|client/option.go` 选标准库 net 时用 `NewSvr/CliTransHandlerFactory` 与 gonet dialer；`remotesvr.Server` 用 `NewTransServerFactory`。
- **下游（标准库 net）**：`ln.Accept` 返回标准 `net.Conn`；`gonetConnExtension` 把它适配为通用 handler。
- **横向**：与 netpoll 共享 `trans.NewDefaultSvr/CliTransHandler` 与 `trans.Extension` 接口，仅 Extension 实现与服务端模型不同。
- **方向**：Accept → go serveConn → OnActive → 循环 OnRead（通用编排+编解码）→ 写回 conn。

## 9. 语言专项适配口径（Go）

- **并发模型**：经典"每连接一 goroutine"模型，简单直观但高并发下 goroutine/栈开销大于 netpoll reactor；Kitex 同时提供两种实现供选型。读超时用 `SetReadDeadline` 手动管理，不依赖事件库。
- **goroutine 生命周期**：每连接 goroutine 与连接同生共死，`serveConn` 退出即 `Close`，边界清晰。
- **适配器模式**：`gonetConnExtension` 把标准库 net.Conn 能力适配为 `trans.Extension`，与 netpollConnExtension 并列——同一通用 handler 框架可插拔不同传输后端。
- **internal 边界**：`pkg/remote/trans` 共享层；`gonet/`、`netpoll/` 平级并列实现。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| gonet 每连接 goroutine 传输架构图 | `gonet-trans-architecture.html` | architecture | showcase |
| Accept/serveConn 处理时序图 | `gonet-trans-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 子目录。Accept→go serveConn→循环 OnRead 时序清晰故补 sequence；无显式业务状态机，不单列 lifecycle。
