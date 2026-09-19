# 流抽象与超时（streaming-core）

> 本文是 `streaming` 域下的叶子子系统文档。域级总览见 `../../streaming.md`。
> 本文只展开「流抽象 Stream/StreamX、流 context、流超时、内部流配置」，不展开客户端/服务端流式 stub 的具体接入（见 `../stream-client-server/stream-client-server.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 旧流接口 `Stream` | gRPC 兼容旧接口：SetHeader/SendHeader/SetTrailer/Header/Trailer/Context/RecvMsg/SendMsg/Close；已 deprecated | `pkg/streaming/streaming.go:29 Stream` |
| 客户端流接口 `ClientStream` | 新流抽象：SendMsg/RecvMsg/Header/Trailer/CloseSend/Context；非并发安全 | `pkg/streaming/streamx.go:76 ClientStream` |
| 服务端流接口 `ServerStream` | 新流抽象：SendMsg/RecvMsg/SetHeader/SendHeader/SetTrailer | `pkg/streaming/streamx.go:103 ServerStream` |
| 泛型流封装 | `NewServerStreamingClient`/`NewClientStreamingClient`/`NewBidiStreamingClient` 把裸 `ClientStream` 包成带类型的泛型 API（Recv/Send 返回 `*Req/*Res`） | `pkg/streaming/streamx.go:135/175/228` |
| 服务端泛型流封装 | `NewServerStreamingServer`/`NewClientStreamingServer`/`NewBidiStreamingServer` 包装 `ServerStream` | `pkg/streaming/streamx.go:155/203/253` |
| 端点请求/响应 `Args`/`Result` | 流调用在 endpoint 链中传递的请求/响应包装，持 ServerStream/ClientStream/兼容旧 Stream | `pkg/streaming/streaming.go:76/84` |
| gRPC 流获取接口 `GRPCStreamGetter` | 服务端从 args 取回底层 gRPC Stream | `pkg/streaming/streaming.go:91` |
| 流超时配置 `TimeoutConfig` | Timeout + DisableCancelRemote（断点续传场景不取消对端） | `pkg/streaming/timeout.go:30` |
| 流超时执行器 `CallWithTimeout` | gopool 跑 f，timer select；超时/ panic 转 `ErrRPCTimeout` 并调 cancel | `pkg/streaming/timeout.go:49` |
| 流结束记录 `FinishStream`/`FinishClientStream` | 手动结束流并记录 stats 事件；流实现需实现 `WithDoFinish` | `pkg/streaming/util.go:43/60` |
| 旧流 ctx 存取 `NewCtxWithStream`/`GetStream` | deprecated 的 ctx 注入旧 Stream | `pkg/streaming/context.go:28/33` |
| 内部流配置 `StreamingConfig` | Recv/Send 中间件切片与 Builder；`BuildRecvInvokeChain`/`BuildSendInvokeChain` 编织到 `stream.RecvMsg/SendMsg` | `internal/stream/stream_option.go:31/57/63` |
| 可取消客户端流 `CancelableClientStream` | gRPC 传输层客户端流实现，`CancelWithErr` 终止本地流并取消对端 | `internal/stream/cancel.go:21` |
| 事件处理器 `EventHandler` | stats 事件回调类型 | `pkg/streaming/streamx.go:271` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Stream`（deprecated） | `pkg/streaming/streaming.go:29` | gRPC 旧流接口，含 `metadata.MD` 头/尾操作 |
| `ClientStream` | `pkg/streaming/streamx.go:76` | 客户端流新接口，方法均带 ctx |
| `ServerStream` | `pkg/streaming/streamx.go:103` | 服务端流新接口 |
| `Header`/`Trailer` | `pkg/streaming/streamx.go:70` | `map[string]string` |
| `ServerStreamingClient[Res]` 等泛型接口 | `pkg/streaming/streamx.go:129/168/221` | 带类型的流式客户端 API，内嵌 `ClientStream` |
| `Args`/`Result` | `pkg/streaming/streaming.go:76/84` | endpoint 流调用的请求/响应包装 |
| `WithDoFinish` | `pkg/streaming/streaming.go:66` | 流实现可选实现，供中间件手动记录流结束；需可重入 |
| `CloseCallbackRegister` | `pkg/streaming/streaming.go:71` | 流关闭回调注册 |
| `TimeoutConfig` | `pkg/streaming/timeout.go:30` | 流 Recv/Send 超时配置 |
| `StreamingConfig`（internal） | `internal/stream/stream_option.go:31` | 流收/发中间件配置与链编织 |
| `CancelableClientStream`（internal） | `internal/stream/cancel.go:21` | gRPC 传输层客户端流的取消接口 |

## 3. 关键调用链

### 链路一：泛型流封装的委托

1. 业务代码拿到底层 `streaming.ClientStream`（由 stream-client-server 叶子创建）后，调 `NewBidiStreamingClient[Req,Res](st)`（`pkg/streaming/streamx.go:228`）返回 `bidiStreamingClientImpl`。
2. 该结构体内嵌 `ClientStream`，`Send(ctx, req)`（`streamx.go:236`）调 `st.SendMsg(ctx, req)`；`Recv(ctx)`（`:240`）`new(Res)` 后调 `st.RecvMsg(ctx, m)` 返回 `(m, err)`。
3. 服务端对称：`NewBidiStreamingServer[Req,Res](st)`（`streamx.go:253`）包装 `ServerStream`，`Recv`/`Send` 委托 `RecvMsg`/`SendMsg`。
4. `ClientStreamingClient.CloseAndRecv`（`streamx.go:187`）先 `CloseSend` 再 `RecvMsg`；`ClientStreamingServer.SendAndClose`（`:216`）只 `SendMsg`。

### 链路二：`CallWithTimeout` 流操作超时

1. 调 `CallWithTimeout(timeout, cancel, f)`（`pkg/streaming/timeout.go:49`）。`timeout<=0` 直接同步跑 `f()`；`cancel==nil` 直接 panic（避免 recv/send 永久阻塞）。
2. `gopool.Go`（`timeout.go:61`）在协程池中跑 `f()`；defer 中 `recover` panic 转 `ErrRPCTimeout` 并 `cancel()`，结果写入带缓冲 `finishChan`。
3. 主 goroutine `select`（`timeout.go:75`）：`timer.C` 到则 `cancel()` 返回 `ErrRPCTimeout`；`finishChan` 到则返回 f 的结果。`finishChan` 容量 1 防止 goroutine 泄漏。

### 链路三：流收发中间件链编织

1. `StreamingConfig.InitMiddlewares(ctx)`（`internal/stream/stream_option.go:39`）把 `RecvMiddlewareBuilders`/`SendMiddlewareBuilders` 用 ctx 实例化。
2. `BuildRecvInvokeChain()`（`:57`）：`endpoint.RecvChain(c.RecvMiddlewares...)(func(stream, resp){ return stream.RecvMsg(resp) })`——最内层直接调 `stream.RecvMsg`。
3. `BuildSendInvokeChain()`（`:63`）对称：最内层 `stream.SendMsg(req)`。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|-------------|------|
| `TimeoutConfig.Timeout` | 0 表示不超时（直接跑 f，不 recover panic） | `pkg/streaming/timeout.go:31/50` |
| `TimeoutConfig.DisableCancelRemote` | 默认 false（超时取消对端防泄漏）；断点续传场景置 true 不取消 | `pkg/streaming/timeout.go:38` |
| `client.WithStreamRecvTimeout` / `WithStreamRecvTimeoutConfig` | 仅 ttstream 生效；Config 优先级高于裸 Timeout | `client/option_stream.go:50/72`（在 client-entry 叶子） |
| `StreamingConfig.Recv/SendMiddlewares` | 由 `WithStreamRecvMiddleware`/`WithStreamSendMiddleware` 注入 | `internal/stream/stream_option.go:32-36` |
| `userStreamNotImplementingWithDoFinish sync.Once` | 未实现 `WithDoFinish` 时只告警一次 | `pkg/streaming/util.go:30` |

## 5. 错误与重试语义

- **流超时**：`CallWithTimeout` 超时返回 `kerrors.ErrRPCTimeout`；panic 也转 `ErrRPCTimeout` 带堆栈。
- **nil cancel 防护**：`cancel==nil` 直接 panic，强制调用方在 ctx 层就传入 cancel，避免 recv/send 永久阻塞导致 goroutine 飙升（`timeout.go:53`）。
- **流结束**：`FinishStream`/`FinishClientStream` 若流未实现 `WithDoFinish`，只 `sync.Once` 告警一次，不报错。
- **非并发安全**：`SendMsg`/`RecvMsg`/`CloseSend` 明确标注 not concurrent-safety，调用方需自行串行化。
- **透传**：`f()` 的业务错误原样返回；`CallWithTimeout` 不重试，只负责超时包裹。

## 6. 并发细节

- **goroutine 边界**：`CallWithTimeout` 用 `gopool.Go`（协程池，外部依赖 `gopkg/util/gopool`）跑 f，主 goroutine select 等待；`finishChan` 容量 1 保证 f 所在 goroutine 不会因阻塞发送而泄漏。
- **非并发安全约定**：所有流方法（SendMsg/RecvMsg/CloseSend）单 goroutine 使用；双向流需两个 goroutine（一收一发，`streamx.go:57-67` 注释）。
- **sync.Once**：`userStreamNotImplementingWithDoFinish` 保证告警只打一次；`WithDoFinish.DoFinish` 要求可重入（建议 `sync.Once`）。
- **context 传播**：流方法均带 `ctx`；`CallWithTimeout` 的 `cancel` 关闭底层传输的 ctx，使阻塞的 recv/send 立即返回。
- **无锁**：本叶子接口层无共享状态锁；状态机在传输层 gRPC/ttstream 实现中（不在本叶子）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/streaming/`：Stream/ClientStream/ServerStream 接口、泛型封装、Args/Result、TimeoutConfig/CallWithTimeout、FinishStream、旧 ctx 存取。
- `internal/stream/`：StreamingConfig 中间件配置、CancelableClientStream。

**Out-of-Scope（不在本仓库源码内）**
- 流状态机的传输层实现：gRPC（`nphttp2/grpc`）、ttstream、netpoll——流接口的具体实现与读写缓冲不在本叶子。
- 客户端/服务端流式 stub 的创建与接入：见 `stream-client-server` 叶子。
- `gopool`（`bytedance/gopkg`）协程池 —— 外部依赖，不在本仓库源码内。
- `metadata.MD`（gRPC metadata）—— 来自 `nphttp2/metadata`，属 gRPC 移植实现。

## 8. 与相邻子系统交互

- 上游 → 本叶子：stream-client-server 叶子创建 `ClientStream`/`ServerStream` 实例，业务代码通过泛型封装 API 使用。
- 本叶子 → 下游：流方法最终调到传输层（gRPC/ttstream）的 `SendMsg`/`RecvMsg` 实现；`CallWithTimeout` 用 `cancel` 关闭传输层 ctx。
- 横向：`StreamingConfig` 的收发中间件由 client/server 的 `StreamOptions` 注入，在 stream-client-server 叶子的收发路径上编织。

## 9. 语言专项适配口径（Go）

- **并发模型**：本叶子是「接口定义 + 泛型封装 + 超时原语」层。流本身是阻塞式消息传递模型（SendMsg/RecvMsg 阻塞直到有数据/缓冲可写），双向流需调用方用两个 goroutine（一收一发）。这是 gRPC 风格的流并发模型，非 K8s Reconcile。
- **goroutine 边界**：唯一显式 goroutine 是 `CallWithTimeout` 的 `gopool.Go`；用带缓冲 channel 配合 timer select 是经典 Go 超时模式。`finishChan` 容量 1 是关键的防泄漏设计。
- **泛型**：Go 1.20 泛型 `[Req,Res any]` 用于把裸流接口包装成带类型的 Recv/Send API，是本叶子的核心 Go 语言特性应用。
- **context 传播**：流方法带 ctx；`CallWithTimeout` 的 cancel 传播到传输层，使阻塞 IO 可取消。
- **internal 边界**：`internal/stream` 隔离 `StreamingConfig`/`CancelableClientStream`，`pkg/streaming` 是公开接口层；依赖方向 `pkg/streaming` ← `internal/stream`（实现引用公开接口）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 流抽象架构图 | `streaming-core-architecture.html` | architecture | standard（showcase 连线穿节点与侧方向校验多次不通过，删边后回退 standard） |
| CallWithTimeout 超时时序 | `streaming-core-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。补一张时序图：`CallWithTimeout` 的「主 goroutine select / gopool 跑 f / timer 超时」三方交互清晰。不补 dataflow（无 ETL 管道，流本身是消息流但由传输层实现）与 lifecycle（流状态机在 gRPC/ttstream 传输层，不在本叶子）。
