# TTStream 多路复用（ttstream-mux）

> 本文是 `transport-implementations` 域下的叶子子系统文档。域级总览见 `../transport-implementations.md`。
> 本文只展开"Kitex 自研流式传输 TTStream 与 netpoll mux 多路复用"，不展开通用收发编排（见 `../remote-transport/trans-handler-pipeline/trans-handler-pipeline.md`）与编解码（见 `../remote-transport/codec-payload/codec-payload.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| TTStream 服务端 handler | 基于 TTHeader 流式的服务端 handler；ProtocolMatch 嗅探流式标记 | `pkg/remote/trans/ttstream/server_handler.go:67` |
| TTStream 客户端 handler | 流式客户端 handler 工厂，可配 option | `pkg/remote/trans/ttstream/client_handler.go:35` |
| 服务端传输/流 | `serverTransport.ReadStream` 读流；`serverStream`/`clientStream` 读写 | `pkg/remote/trans/ttstream/transport_server.go`、`stream_server.go`、`stream_client.go` |
| 客户端连接池 | 长连接/mux 连接/短连接三种客户端传输池 | `pkg/remote/trans/ttstream/client_trans_pool_longconn.go`、`client_trans_pool_muxconn.go`、`client_trans_pool_shortconn.go` |
| 帧处理 | `frame.go`/`frame_handler.go` 流式帧编解码与分发 | `pkg/remote/trans/ttstream/frame.go`、`frame_handler.go` |
| netpoll mux 客户端连接 | `muxCliConn`：基于 netpoll mux 的多路复用连接，按 seqID 分发 | `pkg/remote/trans/netpollmux/mux_conn.go:55` |
| mux 连接池/传输 | `mux_pool.go`/`mux_transport.go`：mux 连接池与传输 | `pkg/remote/trans/netpollmux/mux_pool.go`、`mux_transport.go` |
| seqID 分片映射 | `shard_map.go`：seqID→notify 分片映射 | `pkg/remote/trans/netpollmux/shard_map.go` |
| 控制帧 | `control_frame.go`：mux 控制帧 | `pkg/remote/trans/netpollmux/control_frame.go` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `svrTransHandler` / `clientTransHandler` | `ttstream/server_handler.go:89` / `client_handler.go:44` | TTStream 收发 handler |
| `serverTransport` / `serverStream` / `clientStream` | `ttstream/transport_server.go` / `stream_server.go` / `stream_client.go` | 流式传输与单流读写 |
| `muxCliConn` / `muxConn` | `netpollmux/mux_conn.go:55` / `muxConn` | netpoll mux 多路复用客户端连接 |
| `shardMap` | `netpollmux/shard_map.go` | seqID→notify 的并发分片映射 |

## 3. 关键调用链

### 3.1 TTStream 服务端流读取

1. `svrTransHandler.ProtocolMatch`（`ttstream/server_handler.go:99`）：要求 conn 为 `netpoll.Connection`，`Reader().Peek(8)`（`:104`）后 `ttheader.IsStreaming(data)`（`:108`）判定流式协议。
2. `OnActive`（`:119`）：`newServerTransport(nconn)`（`:121`），注册 `SetOnDisconnect` 关闭 transport。
3. `OnRead`（`:131`）：循环 `trans.ReadStream(ctx)`（`:148`）读到流 → `gofunc.GoFunc`（`:154`）起 goroutine 调 `OnStream` 处理；defer `wg.Wait()` 等全部流结束。

### 3.2 netpoll mux 客户端分发

1. `newMuxCliConn`（`netpollmux/mux_conn.go:43`）：包装 `netpoll.Connection`，`SetOnRequest(c.OnRequest)`（`:48`），`AddCloseCallback(forceClose)`（`:49`）。
2. `muxCliConn.OnRequest`（`:66`）：`parseHeader(connection.Reader())`（`:68`）解析 length/seqID，按 seqID 经 `seqIDMap`（`shardMap`）分发到对应等待者。

### 3.3 帧与流读写

- `frame.go` 定义流式帧；`frame_handler.go` 按帧类型分发；`stream_reader.go`/`stream_writer.go` 实现单流读写缓冲。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `ClientHandlerOption` | 客户端 handler 选项 | `ttstream/client_handler_option.go` |
| `ServerHandlerOption` | 服务端 handler 选项 | `ttstream/server_handler_option.go` |
| `mux.ShardSize` | seqID 分片数 | `netpollmux/mux_conn.go:46` |
| `defaultCodec` | netpollmux 默认 codec | `netpollmux/mux_conn.go:41` |

## 5. 错误与重试语义

- **协议不匹配**：`ProtocolMatch` 非 netpoll 连接或非流式标记返回 `errProtocolNotMatch`（`server_handler.go:102/109`），由 detection 层降级。
- **连接关闭**：`muxCliConn.forceClose`（`mux_conn.go:50`）强制关闭；`closing` 标志标记服务端将关。
- **流 EOF**：`OnRead` 把 `io.EOF` 转 nil（`server_handler.go:140-142`）视为正常结束。
- 本层不负责 RPC 重试。

## 6. 并发细节

- **goroutine 边界**：TTStream 每流一个 goroutine（`gofunc.GoFunc`，`server_handler.go:154`）；`wg.Wait()`（`:138`）等全部流结束后才允许连接 buffer 回收——保证流处理完整。
- **多路复用**：netpollmux 单 TCP 连接承载多并发请求，按 seqID 在 `shardMap`（分片，降低锁竞争）上分发响应。
- **回调注册**：`SetOnRequest`/`AddCloseCallback`/`SetOnDisconnect` 由 netpoll 事件触发。
- **context 传播**：流式 metadata 经 `context.go`/`metadata.go` 透传；每流独立 ctx。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/trans/ttstream/` 自研流式传输全套；`pkg/remote/trans/netpollmux/` 基于 netpoll mux 的多路复用。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/cloudwego/netpoll` 及其 `mux` 包（事件循环、mux 多路复用原语）：外部库。
- 通用收发编排与编解码见对应叶子。

## 8. 与相邻子系统交互

- **上游**：流式 RPC（bidirectional/client/server streaming）使用 TTStream；detection 层 `ProtocolMatch` 嗅探后路由到本叶子。
- **下游（外部 netpoll/mux）**：`serverTransport`/`muxCliConn` 基于 `netpoll.Connection`；netpollmux 复用 netpoll codec 与事件。
- **横向**：依赖 ttheader 标记判定流式（codec-payload/transmeta）；netpollmux 复用 netpoll 传输扩展。
- **方向**：netpoll 连接 → ReadStream/OnRequest → 流 goroutine → 编解码 → 业务。

## 9. 语言专项适配口径（Go）

- **并发模型**：TTStream 每流一 goroutine，用 `sync.WaitGroup` 串联连接级与流级生命周期——连接 OnRead 返回前必须等所有流结束，避免 buffer 提前回收。netpollmux 单连接多路复用 + shardMap 分片降低锁竞争。
- **多路复用**：与 HTTP2（nphttp2）思路类似但走 Kitex 自研 TTHeader 帧；netpollmux 直接复用 netpoll 库的 mux 能力。
- **适配器/边界**：`ttstream`、`netpollmux` 平级并列，均实现 `remote.ServerTransHandlerFactory`；`internal/` 子包为内部辅助。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| TTStream/mux 多路复用架构图 | `ttstream-mux-architecture.html` | architecture | showcase |
| 流读取分发时序图 | `ttstream-mux-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 子目录。流读取→goroutine 处理时序清晰故补 sequence；mux 帧状态机在外部 netpoll/mux 库内，本层不单列 lifecycle。
