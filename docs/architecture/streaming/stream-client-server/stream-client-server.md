# 客户端与服务端流式接入（stream-client-server）

> 本文是 `streaming` 域下的叶子子系统文档。域级总览见 `../streaming.md`。
> 本文只展开「客户端流式 stub（StreamX/Stream）、服务端流式处理（wrapStreamMiddleware）、双向流接入与流生命周期管理」，不展开流接口定义本身（见 `../streaming-core/streaming-core.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 客户端流式入口 `Streaming` 接口 | 生成代码调用；`Stream`（deprecated，<v0.13.0 兼容）与 `StreamX`（新） | `client/stream.go:44 Streaming` |
| `kClient.Stream` | 旧入口：initRPCInfo → sEps → 拿 ClientStream 塞进 `*streaming.Result` | `client/stream.go:52 Stream` |
| `kClient.StreamX` | 新入口：initRPCInfo → sEps → 直接返回 `streaming.ClientStream` | `client/stream.go:116 StreamX` |
| 流式端点工厂 `invokeStreamingEndpoint` | 建 transHandler，织 sendEP/recvEP（新接口）与 grpcSendEP/grpcRecvEP（旧接口），`remotecli.NewStream` 建连接流 | `client/stream.go:163` |
| 客户端流包装 `stream` | 内嵌 `ClientStream`，持 scm/ri/recv/send/timeout 配置；实现 `GRPCStreamGetter`/`WithDoFinish` | `client/stream.go:204 stream`、`:227 newStream` |
| RecvMsg 超时包裹 | `recvWithTimeout` 在 gRPC 且配置超时>0 时走 `callWithTimeout` | `client/stream.go:295 recvWithTimeout`、`:432 callWithTimeout` |
| 流结束幂等 `DoFinish` | `atomic.SwapUint32` 保证只一次；ReleaseConn + TracerCtl.DoFinish | `client/stream.go:340 DoFinish` |
| 旧 gRPC 流包装 `grpcStream` | 包装旧 `Stream`，Recv/Send 走 grpc 中间件链，回调 `st.DoFinish` | `client/stream.go:373 grpcStream`、`:362 newGRPCStream` |
| 服务端流中间件 `wrapStreamMiddleware` | 把 ServerStream 包成带收发中间件链的 `*stream`，替换 Args 中的流 | `server/stream.go:28 wrapStreamMiddleware`、`:48 newStream` |
| 服务端 `stream` | 内嵌 `ServerStream`，RecvMsg/SendMsg 走 recvEP/sendEP + stats 事件 | `server/stream.go:69 stream`、`:90/103` |
| 旧 gRPC 兼容服务端流 `grpcStream` | 服务端对称包装旧 `Stream` | `server/stream.go:123 grpcStream`、`:115 newGRPCStream` |
| ctx 重写 `contextStream` | 包装流并重写 `Context()` 返回中间件处理后的 ctx | `server/stream.go:147 contextStream` |
| gRPC 兼容流 `gRPCCompatibleServerStream` | 保留用户中间件 wrap 过的旧 Stream，避免扩展丢失 | `server/stream.go:167` |
| 流客户端选项 `streamclient.Option` | 全部 deprecated，转发到 `client.Option`；streamx 启用后用 client 包 API | `client/streamclient/definition.go:24`、`client_option.go:41` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Streaming` 接口 | `client/stream.go:44` | 生成代码用的流式客户端入口 |
| `kClient.stream` | `client/stream.go:204` | 客户端流包装，实现 `ClientStream`/`GRPCStreamGetter`/`WithDoFinish` |
| `kClient.grpcStream` | `client/stream.go:373` | 旧 gRPC 流兼容包装 |
| `remotecli.NewStream` | `client/stream.go:186` | 传输层建流连接（StreamConnManager） |
| `server.stream` | `server/stream.go:69` | 服务端流包装，实现 `ServerStream`/`GRPCStreamGetter` |
| `server.grpcStream` | `server/stream.go:123` | 服务端旧 gRPC 流兼容包装 |
| `contextStream` | `server/stream.go:147` | 重写 `Context()` 的装饰器 |
| `gRPCCompatibleServerStream` | `server/stream.go:167` | 保留用户 wrap 的旧 Stream |
| `recvEndpoint`/`sendEndpoint` | `client/stream.go:473/476` | 新接口最内层：直接 `stream.RecvMsg/SendMsg` |
| `streamclient.Option` | `client/streamclient/definition.go:24` | 旧流式客户端选项（deprecated） |

## 3. 关键调用链

### 链路一：客户端发起流（StreamX）

1. 业务代码调 `StreamX(ctx, method)`（`client/stream.go:116`）。检查 inited/closed/ctx 非空。
2. `initRPCInfo`（`:127`）建 RPCInfo，`TracerCtl.DoStart` 开 tracing。defer `recover` panic 转 `ClientPanicToErr`。
3. 取 `MethodInfo()`，校验 `StreamingMode` 非 None/Unary（否则 `ErrNotStreamingMethod`）。
4. `kc.sEps(ctx)`（`:156`）调流式中间件链，最终到 `invokeStreamingEndpoint`（`client/stream.go:163`）。
5. `invokeStreamingEndpoint`：建 transHandler（优先 `GRPCStreamingCliHandlerFactory`，否则 `CliHandlerFactory`）；织 sendEP/recvEP（新）与 grpcSendEP/grpcRecvEP（旧）；`remotecli.NewStream(ctx, ri, handler, RemoteOpt)` 建连接流。
6. `newStream`（`:227`）包装：内嵌 ClientStream，读 `StreamRecvTimeoutConfig`；若底层实现 `GRPCStreamGetter` 则建 `grpcStream`；若实现 `CloseCallbackRegister` 注册 `DoFinish` 为关闭回调。返回 clientStream。

### 链路二：客户端 RecvMsg 超时与流结束

1. 业务 `RecvMsg(ctx, m)`（`client/stream.go:265`）：若有自定义中间件，把流的 ri 重新挂到 ctx（防止中间件用错 ri）。
2. `recvWithTimeout`（`:295`）：非 gRPC 或超时≤0 直接 `s.recv`；否则 `callWithTimeout(tmCfg, f, s.cancel)`。
3. `callWithTimeout`（`:432`）：gopool 跑 f，timer select；超时返回 `codes.RecvDeadlineExceeded`；`DisableCancelRemote=false` 时 `cancel(err)` 调 `CancelableClientStream.CancelWithErr` 终止流。
4. 成功后从 `ri.Invocation().BizStatusErr()` 取业务错误；`handleStreamRecvEvent` 记事件；出错或 client-streaming 模式时 `DoFinish`。
5. `DoFinish`（`:340`）：`atomic.SwapUint32(&s.finished, 1)` 幂等；非 RPC 错误归 nil；`scm.ReleaseConn(err, ri)` 释放连接；`TracerCtl.DoFinish`。

### 链路三：服务端包装流

1. `wrapStreamMiddleware()`（`server/stream.go:28`）返回中间件：进入时从 `s.opt.StreamOptions.BuildSendChain/BuildRecvChain` 与 `s.opt.Streaming.BuildSend/RecvInvokeChain` 织链。
2. 若 req 是 `*streaming.Args`，`newStream`（`server/stream.go:48`）把 `st.ServerStream` 包成 `*stream`（持 recvEP/sendEP），写回 `st.ServerStream` 与 `st.Stream`。
3. 业务 handler 经 `stream.RecvMsg`（`server/stream.go:90`）→ `s.recv(ctx, ServerStream, m)` → `handleStreamRecvEvent`；`SendMsg`（`:103`）对称。
4. 若底层实现 `GRPCStreamGetter`，另建 `grpcStream`（`:115`）走旧 gRPC 中间件链。

## 4. 配置项

| option / 配置 | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `client.WithStreamRecvTimeout` / `WithStreamRecvTimeoutConfig` | 仅 ttstream 生效；Config 优先级高于裸 Timeout | client/option_stream.go（client-entry 叶子） |
| RPCInfo 的 `StreamRecvTimeoutConfig` | newStream 读取；Timeout>0 且 gRPC 才启用 recv 超时 | `client/stream.go:230/296` |
| `TimeoutConfig.DisableCancelRemote` | 超时是否取消对端（断点续传场景 true） | `client/stream.go:450` |
| `RemoteOpt.GRPCStreamingCliHandlerFactory` | 优先用 gRPC 流式 transHandler 工厂 | `client/stream.go:166` |
| `streamclient.Option` 全量 | deprecated，转发 `client.Option`；streamx 启用后用 client 包 | `client/streamclient/client_option.go:41-128` |
| `finished uint32` | 原子标记，DoFinish 幂等 | `client/stream.go:217/341` |

## 5. 错误与重试语义

- **流结束自动**：RecvMsg/SendMsg 出错、EOF、client-streaming 模式收完，自动 `DoFinish`。
- **recv 超时**：`callWithTimeout` 超时返回 `codes.RecvDeadlineExceeded`；默认 cancel 对端防 goroutine 泄漏；`DisableCancelRemote=true` 时不取消（断点续传）。
- **panic**：`callWithTimeout` gopool 内 recover panic 转 `codes.Internal` 并 cancel；`Stream/StreamX` 外层 defer recover 转 `ClientPanicToErr`。
- **BizStatusErr**：服务端业务状态错误存在 RI，客户端 RecvMsg 成功后从 RI 取出返回，不经过 DoFinish。
- **幂等**：DoFinish 用原子交换保证只执行一次 ReleaseConn/DoFinish。
- **不重试**：流一旦错误即结束，不自动重连。

## 6. 并发细节

- **goroutine 边界**：`callWithTimeout` 用 `gopool.Go` 跑 recv；双向流业务侧需两 goroutine（一收一发）。`DoFinish` 由关闭回调或 Recv/Send 错误触发。
- **原子操作**：`finished uint32` 用 `atomic.SwapUint32` 保证 DoFinish 幂等（防 Recv 错误与 CloseCallback 并发触发两次）。
- **关闭回调注册**：`newStream` 若底层实现 `CloseCallbackRegister`，注册 `DoFinish` 为流关闭回调，使传输层主动关闭时也能释放连接。
- **context 传播**：Recv/Send 时若检测到中间件把流的 ri 与调用 ctx 的 ri 不一致，重新 `NewCtxWithRPCInfo` 挂回流的 ri（防中间件用错 ri）。
- **非并发安全**：SendMsg/RecvMsg/CloseSend 单 goroutine；双向流靠业务拆分收发 goroutine。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `client/stream.go`：Stream/StreamX 入口、stream/grpcStream 包装、DoFinish、callWithTimeout。
- `server/stream.go`：wrapStreamMiddleware、服务端 stream/grpcStream、contextStream、gRPCCompatibleServerStream。
- `client/streamclient/`：旧流式客户端选项（deprecated 转发层）。

**Out-of-Scope（不在本仓库源码内）**
- 传输层流连接：`remotecli.NewStream`/`StreamConnManager`、gRPC/ttstream 的实际读写与帧协议。
- 流收发中间件链的具体成员（用户中间件）：由 StreamOptions 注入，不在本叶子。
- `gopool` 协程池 —— 外部依赖，不在本仓库源码内。
- 对端服务、netpoll —— 不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游 → 本叶子：client-entry 叶子的 `kClient.sEps`（流式端点链）调到 `invokeStreamingEndpoint`；server-entry 的 `buildMiddlewares` prepend `wrapStreamMiddleware`。
- 本叶子 → 下游：`remotecli.NewStream` 建流连接（传输层）；`StreamOptions.BuildRecv/SendChain` 织收/发中间件（streaming-core 叶子定义的 StreamingConfig 机制）；`TracerCtl` 记 stats 事件。
- 横向：复用 streaming-core 叶子的 `ClientStream`/`ServerStream`/`TimeoutConfig`/`WithDoFinish` 接口。

## 9. 语言专项适配口径（Go）

- **并发模型**：本叶子是「流生命周期管理 + 超时原语」。流本身是阻塞式消息传递；双向流靠业务双 goroutine。`callWithTimeout` 复用 gopool+timer select 模式（与 streaming-core 的 CallWithTimeout 同源，但这里用 gRPC status 错误码）。
- **goroutine 生命周期**：唯一显式 goroutine 是 `callWithTimeout` 的 gopool；`CloseCallbackRegister` 把 `DoFinish` 交给传输层关闭事件回调，避免 goroutine 泄漏。
- **原子操作**：`finished uint32` 是典型「幂等关闭」模式——用 `atomic.SwapUint32` 替代 mutex，因为只需保证 DoFinish 一次且无复杂临界区。
- **context 传播**：Recv/Send 时重挂 RI 是「中间件可能污染 ctx」的防御性设计；cancel 经 `CancelableClientStream` 传到传输层。
- **internal 边界**：`internal/stream.CancelableClientStream` 由 gRPC 传输层实现，本叶子通过类型断言调用，不依赖具体实现。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 流式接入架构图 | `stream-client-server-architecture.html` | architecture | showcase |
| StreamX 建流时序 | `stream-client-server-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。补一张时序图：StreamX 从入口到建流包装的主路径清晰。不补 dataflow（流消息流在传输层）与 lifecycle（流状态机在 gRPC/ttstream）。
