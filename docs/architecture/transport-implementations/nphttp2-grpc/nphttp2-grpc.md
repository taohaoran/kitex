# HTTP2/gRPC 传输移植实现（nphttp2-grpc）

> 本文是 `transport-implementations` 域下的叶子子系统文档。域级总览见 `../transport-implementations.md`。
> 本文只展开"gRPC over HTTP2 传输：Kitex 集成 handler、连接/流封装、移植的 http2/grpc 传输栈、gRPC 连接池"，不展开通用收发编排（见 `../remote-transport/trans-handler-pipeline/trans-handler-pipeline.md`）与编解码（见 `../remote-transport/codec-payload/codec-payload.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 服务端/客户端 handler 工厂 | gRPC 传输的 Kitex 集成层 handler，用 `grpc.NewGRPCCodec` | `pkg/remote/trans/nphttp2/server_handler.go:58` / `client_handler.go:33` |
| HTTP2 客户端前言嗅探 | `ProtocolMatch`：peek HTTP2 client preface 判定协议是否 gRPC | `pkg/remote/trans/nphttp2/server_handler.go:97` |
| 服务端连接封装 | `serverConn`：把 `grpc.ServerTransport`+`grpc.Stream` 包成 `net.Conn`，提供 ReadFrame/WriteFrame | `pkg/remote/trans/nphttp2/server_conn.go:42` |
| 客户端连接封装 | `clientConn`：把 `grpc.ClientTransport`+`grpc.Stream` 包成 `net.Conn`，携带 metadata/compress | `pkg/remote/trans/nphttp2/client_conn.go:81` |
| 流处理 | `handleFunc` 处理每个 gRPC stream，按 unary/server/client/bidi 分派 | `pkg/remote/trans/nphttp2/server_handler.go:149` |
| 流式封装 | `stream.go` 实现 Kitex `streaming.ServerStream/ClientStream` | `pkg/remote/trans/nphttp2/stream.go` |
| gRPC 连接池 | `conn_pool.go`/`conn_pool_slot.go`：gRPC 长连接复用 | `pkg/remote/trans/nphttp2/conn_pool.go` |
| 移植 http2/grpc 传输栈 | `grpc/` 子包：HTTP2 framer、flow control、controlbuf、client/server transport、keepalive、bdp 估计 | `pkg/remote/trans/nphttp2/grpc/transport.go`、`http2_client.go`、`http2_server.go`、`controlbuf.go`、`flowcontrol.go`、`framer.go` |
| gRPC status/codes/metadata | `status/`、`codes/`、`metadata/`、`peer/` 适配 | `pkg/remote/trans/nphttp2/{codes,meta_api,peer}.go` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `svrTransHandler` / `cliTransHandler` | `server_handler.go:77` / `client_handler.go` | Kitex 集成 handler；持有 `grpc.Codec` 与活跃 stream 链表 |
| `serverConn` | `server_conn.go:42` | 服务端流到 `net.Conn` 的适配器；`ReadFrame`/`WriteFrame` 读写 HTTP2 DATA/HEADERS 帧 |
| `clientConn` | `client_conn.go:81` | 客户端流到 `net.Conn` 的适配器；`Header`/`Trailer`/`GetRecvCompress` |
| `grpc.Stream` / `grpc.ServerTransport` / `grpc.ClientTransport` | `grpc/transport.go` | 移植的 gRPC 传输抽象（流、服务端/客户端传输） |
| `grpc.ClientPreface` | `grpc/` | HTTP2 客户端前言常量，用于 `ProtocolMatch` |

## 3. 关键调用链

### 3.1 服务端 gRPC 流处理

1. `newSvrTransHandler`（`server_handler.go:66`）：`codec = grpc.NewGRPCCodec(WithThriftCodec(opt.PayloadCodec))`（`:70`），维护活跃 stream 链表 `li`。
2. `ProtocolMatch`（`:97`）：`npReader.Peek(prefaceReadAtMost)`（`:103`）比对 `grpcTransport.ClientPreface`，判定是否 gRPC。
3. `OnRead`（`:134`）→ `handleFunc(s *grpcTransport.Stream, svrTrans, conn)`（`:149`）按流类型分派 unary/streaming。
4. `serverConn.ReadFrame`（`server_conn.go:49`）从 `grpc.Stream` 读帧；`WriteFrame`（`:89`）写回。

### 3.2 客户端 gRPC 调用

1. `newClientConn`（`client_conn.go:81`）：基于 `grpc.ClientTransport` 与 `addr` 建客户端流。
2. `cliTransHandler.Write`（`client_handler.go:55`）→ `codec.Encode` 经 `clientConn.WriteFrame` 发 HTTP2 帧。
3. `cliTransHandler.Read`（`:65`）→ `clientConn.ReadFrame` 读响应；`Header`/`Trailer` 取 metadata。

### 3.3 移植传输栈

- `grpc/transport.go` 实现 HTTP2 连接与流状态机；`controlbuf.go` 控制帧收发缓冲；`flowcontrol.go` 实现 HTTP2 流级/连接级流控；`framer.go` 读写 HTTP2 帧；`bdp_estimator.go` 带宽延迟积估计。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `prefaceReadAtMost` | 嗅探前言最多读 min(ClientPreface, 8 字节) | `server_handler.go:88` |
| `WithThriftCodec` | gRPC codec 内嵌 Thrift payload codec | `server_handler.go:70` |
| `parseGraceAndPollTime` | 优雅关闭的 grace/poll 时间 | `server_handler.go:456` |
| gRPC keepalive | `grpc/keepalive.go` 配置 | `pkg/remote/trans/nphttp2/grpc/keepalive.go` |

## 5. 错误与重试语义

- **协议不匹配**：`ProtocolMatch` 比对失败返回 `error protocol not match`（`server_handler.go:112`），由 detection 层降级到其他协议。
- **流错误**：gRPC status 经 `status/`、`codes/` 适配为 Kitex `kerrors`；`OnError`（`server_handler.go:398`）记录。
- **panic 兜底**：`handleFunc` 内 recover 并 `finishTracer` 记录（`server_handler.go:478`）。
- 本层不负责 RPC 重试；gRPC 连接失败由 client 层处理。

## 6. 并发细节

- **goroutine 边界**：每个 gRPC stream 由 `handleFunc` 在独立 goroutine 处理（`server_handler.go:149`）；HTTP2 多路复用使单 TCP 连接承载多并发流。
- **活跃流管理**：`svrTransHandler.li *list.List` 用 `sync.Mutex`（`:83`）维护活跃 stream，供优雅关闭遍历。
- **流控**：HTTP2 流级/连接级窗口由 `flowcontrol.go` 管理（移植自 x/net/http2），背压经窗口更新帧传播。
- **context 传播**：gRPC metadata 经 `metadata.FromIncomingContext/OutgoingContext` 透传（见 transmeta 叶子）；RPC 超时经 gRPC deadline。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/trans/nphttp2/` 根层 Kitex 集成 handler、连接封装、流、连接池、status/codes/metadata 适配。
- `pkg/remote/trans/nphttp2/grpc/` 移植的 HTTP2/gRPC 传输栈（已 vendored 进本仓库）。

**Out-of-Scope（不在本仓库源码内）**
- 该 `grpc/` 子包移植自 `golang.org/x/net/http2` 与 `google.golang.org/grpc`（上游参考实现，已复制进本仓库修改）。
- netpoll 事件库（外部）；通用收发编排与编解码见对应叶子。

## 8. 与相邻子系统交互

- **上游**：`internal/server|client/option.go` 选 gRPC/HTTP2 传输时用 `NewSvr/CliTransHandlerFactory`；detection 层先 `ProtocolMatch` 嗅探。
- **下游（移植 grpc 栈）**：`serverConn/clientConn` 包装 `grpc.Stream/Transport`；`grpc/` 子包实现 HTTP2 帧与流控。
- **横向**：`codec/grpc` 子包提供 gRPC 压缩与 codec；transmeta 的 HTTP2 handler 经 metadata 透传。
- **方向**：HTTP2 帧 → grpc.Stream → serverConn(net.Conn) → Kitex handler → codec → 业务。

## 9. 语言专项适配口径（Go）

- **并发模型**：HTTP2 多路复用 + 每流一 goroutine，单连接高并发；流控窗口在 `flowcontrol.go` 以 channel/信号量实现背压。
- **移植/边界**：`grpc/` 子包是对 x/net/http2 与 grpc-go 的 vendored 移植（非外部依赖），保持上游 API 形态；根层 `nphttp2` 负责把它适配进 Kitex 的 `remote.TransHandler`/`net.Conn` 抽象——适配器模式。
- **internal 边界**：`nphttp2` 为公开传输实现包；`grpc/` 为其内部依赖子包。
- **metadata 透传**：gRPC 用 HTTP2 HEADERS 帧携带 metadata，与 Kitex `TransInfo` 经 transmeta 双向转换。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| gRPC/HTTP2 传输分层架构图 | `nphttp2-grpc-architecture.html` | architecture | showcase |
| gRPC 流处理时序图 | `nphttp2-grpc-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 子目录。流处理时序清晰故补 sequence；HTTP2 帧状态机已在移植上游栈内，本层不单列 lifecycle。
