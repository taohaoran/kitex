# 收发处理管道与消息抽象（trans-handler-pipeline）

> 本文是 `remote-transport` 域下的叶子子系统文档。域级总览见 `../remote-transport.md`。
> 本文只展开"收发处理管道、TransHandler 接口、消息抽象"的职责边界，不重复展开具体传输实现（netpoll/gonet/nphttp2 等，见 `transport-implementations` 域各叶子）与编解码细节（见 `../codec-payload/codec-payload.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| TransHandler 接口族 | 定义客户端/服务端收发处理器抽象（Read/Write/OnMessage/OnActive/OnInactive/OnError），是 Netty 式 handler 角色 | `pkg/remote/trans_handler.go` |
| 客户端工厂接口 | `ClientTransHandlerFactory.NewTransHandler(*ClientOption)` 创建客户端 handler | `pkg/remote/trans_handler.go:29` |
| 服务端工厂接口 | `ServerTransHandlerFactory.NewTransHandler(*ServerOption)` 创建服务端 handler | `pkg/remote/trans_handler.go:34` |
| TransReadWriter | 最基础的收发读写抽象（Write/Read 操作 `net.Conn` 与 `Message`） | `pkg/remote/trans_handler.go:39` |
| InvokeHandleFuncSetter | 允许外部把业务调用函数 `endpoint.Endpoint` 注入 handler（服务端把 handler 与业务 invoke 解耦） | `pkg/remote/trans_handler.go:67` |
| GracefulShutdown | 优雅关闭连接的扩展接口 | `pkg/remote/trans_handler.go:72` |
| ClientStreamFactory | 客户端流创建扩展点 | `pkg/remote/trans_handler.go:78` |
| TransPipeline 管道 | Netty 式责任链：串联 `inboundHdrls`（入站）+ `outboundHdrls`（出站）+ 末端 `netHdlr`（真正读写网络） | `pkg/remote/trans_pipeline.go:49` |
| Inbound/Outbound/Duplex handler | 绑定到连接的入站/出站/双向 handler 抽象（用于限流、元信息透传等跨切关注点） | `pkg/remote/trans_pipeline.go:28/34/43` |
| TransServer 抽象 | 远程服务端：CreateListener/BootstrapServer/Shutdown/ConnCount | `pkg/remote/trans_server.go:31` |
| Message 消息抽象 | Kitex 消息的核心抽象（RPCInfo/Data/MessageType/TransInfo/PayloadCodec/Tags/回收） | `pkg/remote/message.go:93` |
| MessageType 枚举 | Call/Reply/Exception/Oneway/Stream/Heartbeat（0-4 对齐 thrift.TMessageType） | `pkg/remote/message.go:43` |
| TransInfo 传输元信息 | 字符串/整型两张透传信息表，带 sync.Pool 复用 | `pkg/remote/message.go:247` |
| ServiceSearcher | 按服务名/方法名在服务端反查 serviceinfo 的扩展点（经 context 传递） | `pkg/remote/message.go:75` |
| 默认服务端 handler | `svrTransHandler`：OnRead 编排 Read→OnMessage→Write，注入业务 invoke、trace/profiler、panic 恢复 | `pkg/remote/trans/default_server_handler.go:52` |
| 默认客户端 handler | `cliTransHandler`：Write 编码发送、Read 解码接收、OnMessage 为空实现 | `pkg/remote/trans/default_client_handler.go:39` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `TransHandler` | `trans_handler.go:46` | 组合 `TransReadWriter` + 生命周期回调（OnInactive/OnError/OnMessage/SetPipeline），是所有收发处理器的根接口 |
| `ServerTransHandler` | `trans_handler.go:60` | 在 `TransHandler` 上增加 `OnActive`（建连）与 `OnRead`（读循环单次），服务端专用 |
| `TransPipeline` | `trans_pipeline.go:49` | 自身实现 `TransHandler`/`ServerTransHandler`，对外是"一个 handler"，对内把事件沿责任链分发；`netHdlr` 为末端真正读写者 |
| `NewTransPipeline(netHdlr)` | `trans_pipeline.go:66` | 构造管道并反向 `netHdlr.SetPipeline(self)`，形成自引用（末端 handler 回调管道发起读写） |
| `Message` | `message.go:93` | 一次 RPC 请求/响应的抽象载体；`Recycle()` 归还 `sync.Pool` 复用 |
| `message`（私有实现） | `message.go:134` | `Message` 默认实现；`zero()` 逐字段清零后 Put 回 `messagePool` |
| `TransInfo` | `message.go:247` | 透传信息表（`strInfo map[string]string` + `intInfo map[uint16]string`），`PutTrans*Info` 支持整表替换或合并 |
| `svrTransHandler` | `default_server_handler.go:52` | 默认服务端 handler；`inkHdlFunc` 由 `SetInvokeHandleFunc` 注入，`transPipe` 由 `SetPipeline` 注入 |
| `cliTransHandler` | `default_client_handler.go:39` | 默认客户端 handler；Write 走 `codec.Encode`，Read 走 `codec.Decode` |
| `MetaHandler` / `StreamingMetaHandler` | `trans_meta.go:24/30` | 元信息读写扩展点（WriteMeta/ReadMeta/OnConnectStream/OnReadStream），由 bound handler 接入管道 |

## 3. 关键调用链

### 3.1 服务端单连接请求处理主链（OnRead）

1. 传输实现（netpoll/gonet）在连接可读时调用管道 `OnRead`，最终落到 `svrTransHandler.OnRead`（`default_server_handler.go:143`）。
2. `newCtxWithRPCInfo`（`:122`）从 per-connection 复用池取出或新建 `rpcinfo.RPCInfo`；若服务处于优雅关闭则给 `ri.To()` 打 `ConnResetTag`。
3. `transPipe.Read`（`:183`）经管道末端 `netHdlr.Read` 解码入站消息 `recvMsg`；Read 内部先 `DecodeMeta` 再 `DecodePayload`（`:103-114`）。
4. 按消息类型分支：心跳（`Heartbeat`）直接构造心跳回复；普通调用构造 `sendMsg`（Oneway 用空 Result）。
5. `transPipe.OnMessage`（`:204`）沿入站链执行后落到 `svrTransHandler.OnMessage`（`:228`）→ `t.inkHdlFunc(ctx, args.Data(), result.Data())`（`:229`）执行业务 handler。
6. `transPipe.Write`（`:219`）沿出站链编码并写出 `sendMsg`。
7. defer 中统一 `RecycleMessage`、reset rpcinfo、`finishTracer`/`finishProfiler`，并对 panic 做 recover 包装为 `ErrPanic`。

### 3.2 管道事件分发（TransPipeline）

- 出站写：`TransPipeline.Write`（`trans_pipeline.go:86`）先依次执行 `outboundHdrls[i].Write`，全部成功后调用 `netHdlr.Write` 真正落网络；任一出站 handler 报错即短路返回。
- 入站读：`OnRead`（`trans_pipeline.go:120`）依次执行 `inboundHdrls[i].OnRead`，再类型断言为 `ServerTransHandler` 调 `netHdlr.OnRead`。
- 入站消息：`OnMessage`（`trans_pipeline.go:145`）依次执行入站链；若 `result.MessageType() == Exception`（`:153`）则**短路**，不再下发给 `netHdlr.OnMessage`（异常回复由上游已处理）。

### 3.3 客户端发送/接收（remotecli 侧）

- `client.Send`（`remotecli/client.go:88`）→ `transHdlr.Write(ctx, conn, req)`；失败时 `connManager.ReleaseConn(err, ri)` 丢弃连接。
- `client.Recv`（`remotecli/client.go:97`）：`resp != nil` 时 `transHdlr.Read` + `transHdlr.OnMessage`；Oneway（`resp==nil`）时 `time.Sleep(500µs)`（`:105`）尽力等待 flush 后再关连接。
- 管道装配点：服务端 `server/server.go:576` 与 `server/invoke.go:96`、客户端 `client/client.go:582` 均调用 `remote.NewTransPipeline(handler)`。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `rpcinfo.PoolEnabled()` | 服务端按连接复用 rpcinfo（OnRead 内延迟 reinit）；关闭则每请求新建 | `default_server_handler.go:124` |
| `ServerOption.TracerCtl` | 为 nil 时兜底为 `&rpcinfo.TraceController{}`（避免单测 panic） | `default_server_handler.go:45` |
| `ServerOption.Profiler` / `ProfilerTransInfoTagging` | 非 nil 时在 DecodeMeta 后打 profiler tag | `default_server_handler.go:105` |
| `inGracefulShutdown`（atomic uint32） | `GracefulShutdown()` 置 1；OnRead 对所有响应打 ConnResetTag 让客户端主动断连 | `default_server_handler.go:59/356` |
| Oneway 回复写 | 服务端 Write 直接返回 nil 不写网络（`default_server_handler.go:73`）；客户端 Oneway 收前 sleep 500µs | |

## 5. 错误与重试语义

- **Read/OnRead panic 恢复**：`Read`（`default_server_handler.go:90`）与 `OnRead`（`:151`）均 `recover()`，包装为 `kerrors.ErrPanic` 并写入 stats；OnRead 出错后 `writeErrorReplyIfNeeded` 尽力回写 `Exception` 消息。
- **远端关闭错误**：`OnError`（`:248`）经 `IsRemoteClosedErr` 识别远端断连，用 `remoteClosedWarn`（指数退避 `logbackoff.Exponential`）打 warn，避免日志洪泛；并给 `ri.From()` 打 `RemoteClosedTag`。
- **协议错误**：Read 解码失败时给 `recvMsg.Tags()[ReadFailed]=true`（`:116`），错误上抛由传输层关闭连接。
- **超时**：客户端 Read 出错经 `ext.IsTimeoutErr` 判定后包装为 `kerrors.ErrRPCTimeout.WithCause`（`default_client_handler.go:78`）。
- **连接回收**：`ReleaseConn`（`conn_wrapper.go:81`）按 err 与 `ConnResetTag`/Oneway 决定 `Put`（复用）或 `Discard`（丢弃）；重试本身不在本层，由 client 侧 retry 中间件负责。

## 6. 并发细节

- **goroutine 边界**：本层不直接起 goroutine；服务端连接读循环由传输实现（netpoll eventloop / gonet 每连接一 goroutine）驱动，`OnRead` 是"单次读处理"的同步入口；`remotesvr.server.Start`（`remotesvr/server.go:66`）用 `gofunc.GoFunc` 起一个 goroutine 跑 `BootstrapServer`。
- **连接级状态复用**：服务端 rpcinfo 按连接复用，`OnActive` 初始化、`OnInactive` 回收（`PutRPCInfo`）、`OnRead` defer reset，避免每请求分配。
- **对象池**：`message`/`transInfo`/`client`/`ConnWrapper` 均用 `sync.Pool` 复用（`message.go:29`、`client.go:39`、`conn_wrapper.go:31`），`Recycle` 必须与 `New` 配对，禁止跨 goroutine 长期持有。
- **原子标志**：`inGracefulShutdown` 用 `atomic.Load/StoreUint32`（`default_server_handler.go:132/357`），无锁读多写少。
- **context 传播**：`TransHandler` 所有方法首参 `context.Context`，贯穿 trace/profiler/超时；`WithServiceSearcher`/`GetServiceSearcher`（`message.go:82/88`）经 ctx 传递服务反查器。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/trans_handler.go`、`trans_pipeline.go`、`trans_server.go`、`message.go`、`trans_meta.go` 的接口与抽象。
- `pkg/remote/trans/default_server_handler.go`、`default_client_handler.go` 的默认收发编排。

**Out-of-Scope（不在本仓库源码内）**
- 具体网络读写与事件驱动：netpoll（外部库，见 `transport-implementations/netpoll-trans`）、标准库 `net`（gonet）、HTTP2/gRPC 栈（nphttp2，移植自 golang.org/x/net/http2，外部）。
- 真正的业务 invoke 与治理中间件编织：由 `server/invoke.go`、`client/client.go` 及 `pkg/endpoint` 负责，本层只持有 `endpoint.Endpoint` 句柄。
- 编解码细节（TTHeader/thrift/pb payload codec、压缩、bytebuf 实现）见 `codec-payload` 叶子，不在本叶子重复。

## 8. 与相邻子系统交互

- **上游（client/server 包）**：`client/client.go:582`、`server/server.go:576` 用 `NewTransPipeline(handler)` 装配管道并 `AddInboundHandler` 注入限流/元信息 bound handler；`SetInvokeHandleFunc` 把业务 `endpoint.Endpoint` 注入服务端 handler。
- **下游（传输实现）**：管道末端 `netHdlr` 由各传输实现提供（netpoll/gonet/nphttp2 的 `server_handler.go`），真正的 `net.Conn` 读写与帧解析在那里。
- **横向**：`bound/transmeta_bound.go`、`bound/limiter_inbound.go` 实现 `InboundHandler` 挂入管道；`codec-payload` 提供 `remote.Codec` 供 handler 在 Read/Write 内调用。
- **方向**：网络字节流 → 传输实现 OnRead → 管道 Read 解码 → 管道 OnMessage 调业务 → 管道 Write 编码 → 传输实现写回对端。

## 9. 语言专项适配口径（Go）

- **并发模型**：非 K8s Reconcile/informer 模式；这是**网络事件驱动 + 同步请求处理**模型——传输实现负责 eventloop/goroutine 调度，本层提供同步的 `OnRead` 单次处理语义。goroutine 启停边界清晰：`remotesvr.Start` 一个 goroutine 跑 BootstrapServer，连接读循环 goroutine 归传输实现，本层无泄漏风险点。
- **context 超时/取消传播**：`context.Context` 作为所有 handler 方法首参贯穿全链，trace/profiler/超时均挂载其上；服务端复用 rpcinfo 时通过 `InitOrResetRPCInfoFunc` 重置而非新建 context。
- **对象池/sync.Pool**：高性能 RPC 框架的关键模式——`message`/`transInfo`/`client`/`ConnWrapper` 池化复用降低 GC；代价是必须严格配对 `New`/`Recycle`，且 `Recycle` 后禁止再引用（`zero()` 会清空所有字段）。
- **internal 边界与依赖方向**：接口定义在 `pkg/remote`（消费方/框架侧），实现在 `pkg/remote/trans/*`（实现侧）——依赖倒置：`remote` 不 import 具体传输实现，传输实现 import `remote` 并注入。`pkg/remote/trans/internal/logbackoff` 为跨传输实现共享的内部辅助包，不可被外部 import。
- **多二进制**：本仓库运行时为库，无 cmd 二进制入口；唯一二进制是代码生成工具 `tool/cmd/kitex`，与本层无关。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 管道装配与职责分层架构图 | `trans-handler-pipeline-architecture.html` | architecture | showcase |
| 服务端 OnRead 请求处理时序图 | `trans-handler-pipeline-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 子目录（`trans-handler-pipeline-architecture.json`、`trans-handler-pipeline-sequence.json`）。本叶子补 dataflow/lifecycle：收发主链为同步请求-响应，已由 sequence 完整表达；连接建/断生命周期由传输实现持有，本层只暴露 OnActive/OnInactive 回调，不单列 lifecycle 图。
