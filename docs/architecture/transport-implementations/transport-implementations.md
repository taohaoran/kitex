# transport-implementations（传输实现）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 域职责

本域是 Kitex 远程通信的"具体传输实现层"：在 `pkg/remote/trans/` 下，为 remote-transport 域定义的抽象（`remote.ServerTransHandlerFactory`、`trans.Extension`、`remote.Dialer`）提供多种可插拔后端。按并发模型与协议分为：netpoll 事件驱动、标准库 net 每连接 goroutine、HTTP2/gRPC 移植栈、TTStream/netpollmux 流式多路复用，以及最外层的协议检测分发与进程内直连 invoke。

核心代码路径：`pkg/remote/trans/{netpoll,gonet,nphttp2,ttstream,netpollmux,detection,invoke,internal}/`。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| netpoll-trans | [netpoll-trans.md](netpoll-trans/netpoll-trans.md) | [架构图](netpoll-trans/netpoll-trans-architecture.html) | [时序图](netpoll-trans/netpoll-trans-sequence.html) | 基于 netpoll 事件循环的传输实现 |
| gonet-trans | [gonet-trans.md](gonet-trans/gonet-trans.md) | [架构图](gonet-trans/gonet-trans-architecture.html) | [时序图](gonet-trans/gonet-trans-sequence.html) | 基于标准库 net 每连接一 goroutine 的传输 |
| nphttp2-grpc | [nphttp2-grpc.md](nphttp2-grpc/nphttp2-grpc.md) | [架构图](nphttp2-grpc/nphttp2-grpc-architecture.html) | [时序图](nphttp2-grpc/nphttp2-grpc-sequence.html) | HTTP2/gRPC 传输移植（含 vendored grpc 栈） |
| ttstream-mux | [ttstream-mux.md](ttstream-mux/ttstream-mux.md) | [架构图](ttstream-mux/ttstream-mux-architecture.html) | [时序图](ttstream-mux/ttstream-mux-sequence.html) | TTStream 流式 + netpollmux 多路复用 |
| detection-invoke | [detection-invoke.md](detection-invoke/detection-invoke.md) | [架构图](detection-invoke/detection-invoke-architecture.html) | [时序图](detection-invoke/detection-invoke-sequence.html) | 协议检测多路分发 + 进程内直连 invoke |

## 3. 域级机制细节

- **两种并发模型并列**：netpoll 用外部事件循环 reactor（OnPrepare 建连、onConnRead 读回调），gonet 用 `go serveConn` 每连接一 goroutine + `SetReadDeadline` 手动控超时——二者共享 `trans.NewDefaultSvr/CliTransHandler`，仅 `trans.Extension` 实现不同。
- **连接扩展适配**：`netpollConnExtension`/`gonetConnExtension` 把各自连接的读超时、Reader/Writer、错误类型适配为通用 `trans.Extension`，使通用 handler 不感知后端差异（依赖倒置）。
- **多协议统一入口**：`detection.svrTransHandler` 包装默认 handler + 多个可检测 handler，建连后一次性 `ProtocolMatch` 嗅探并把结果存入连接 ctx 的 `handlerWrapper`，后续请求直接委托——一连接只检测一次。
- **多路复用**：nphttp2 与 ttstream/netpollmux 均在单 TCP 连接上承载多并发流，分别走 HTTP2 帧与 TTHeader 帧；流级 goroutine 由 `WaitGroup` 串联连接生命周期。

## 4. 域级图（可选）

本域未单独产出域级架构图；各叶子架构图已覆盖各后端组件与边界，域级关系见上方叶子索引。
