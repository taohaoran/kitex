# streaming（流式）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 域职责

streaming 域负责 Kitex 的流式 RPC（gRPC 风格与 ttstream）。`pkg/streaming` 定义流抽象接口（ClientStream/ServerStream/旧 Stream）与泛型流封装、流超时原语；`client/stream.go` 与 `server/stream.go` 在客户端/服务端把底层流包装成带收发中间件链与生命周期管理的流实现，支撑四种流式模式（一元/客户端流/服务端流/双向流）。流状态机本身在 gRPC/ttstream 传输层，不在本域。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| streaming-core | [streaming-core.md](streaming-core/streaming-core.md) | [架构图](streaming-core/streaming-core-architecture.html) | [时序图](streaming-core/streaming-core-sequence.html) | 流抽象 Stream/StreamX、流 context、流超时 CallWithTimeout、内部流配置 StreamingConfig |
| stream-client-server | [stream-client-server.md](stream-client-server/stream-client-server.md) | [架构图](stream-client-server/stream-client-server-architecture.html) | [时序图](stream-client-server/stream-client-server-sequence.html) | 客户端流式 stub（StreamX/Stream）、服务端流式处理（wrapStreamMiddleware）、双向流接入与 DoFinish 生命周期 |

## 3. 域级机制细节

- **四种流式模式**：一元（Req→Res）、客户端流（stream Req→Res）、服务端流（Req→stream Res）、双向流（stream Req↔stream Res，需收发两个 goroutine）。
- **双接口演进**：旧 `Stream`（gRPC 兼容，带 `metadata.MD`）与新 `ClientStream`/`ServerStream`（方法带 ctx）并存；泛型封装 `NewBidiStreamingClient` 等把裸流包成带类型 API。
- **流超时**：`CallWithTimeout`/`callWithTimeout` 用 gopool + timer select 包裹 Recv；`TimeoutConfig.DisableCancelRemote` 支持断点续传（超时不取消对端）。
- **流结束幂等**：客户端 `DoFinish` 用 `atomic.SwapUint32` 保证只一次 ReleaseConn + TracerCtl.DoFinish；`CloseCallbackRegister` 把 DoFinish 注册为传输层关闭回调。
- **收发中间件链**：`StreamingConfig.BuildRecv/SendInvokeChain` 把用户收发中间件织到最内层 `stream.RecvMsg/SendMsg`；服务端 `wrapStreamMiddleware` 在请求进入时把 ServerStream 替换为带链包装。

## 4. 域级图（可选）

本域未单独出域级架构图；两张叶子图已覆盖流抽象层与接入层两个视角，见叶子索引表。
