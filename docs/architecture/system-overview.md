# Kitex 系统架构总览

> 本文基于 CloudWeGo Kitex 源码深度分析产出。源码基准：`github.com/cloudwego/kitex`，Go 1.20，git commit `4fffa48`，约 756 个非测试 Go 文件 / 约 8.5 万行。
> 所有图表由 archify 渲染为自包含交互式 HTML。

## 1. 项目概述

Kitex 是 CloudWeGo（字节跳动开源）的**高性能、强可扩展 Go RPC 框架**，CNCF 景观项目，Apache-2.0 协议。它本身是一个被业务 `import` 的**库**（而非多二进制服务），唯一独立二进制是代码生成工具 `tool/cmd/kitex`。

| 项 | 值 |
|---|---|
| module | `github.com/cloudwego/kitex` |
| Go 版本 | go 1.20 |
| 非测试 Go 文件 | 约 756 个 |
| 非测试代码行数 | 约 85,004 行 |
| 独立二进制入口 | `tool/cmd/kitex/main.go`（代码生成） |
| 主语言 | Go（无 cgo 核心；netpoll 为外部网络库） |

架构范式：**接口 + 默认实现**的可注入式框架。用户通过 `client.NewClient` / `server.NewServer` 构造端点，治理能力（注册发现、负载均衡、重试、熔断、限流、观测）以 `endpoint.Middleware` 形式编织进调用链，传输层以 `Trans` 接口抽象（netpoll / gonet / nphttp2 / ttstream 多实现可换）。

## 2. 功能总览

| 领域 | 能力 | 归属域 / 叶子 |
|---|---|---|
| 客户端运行时 | NewClient 入口、配置装载、泛化客户端、callopt | client-core |
| 客户端中间件 | 中间件链编织、rpcinfo/context 透传、RPCTimeout | client-core |
| 服务端运行时 | NewServer 入口、服务注册、泛化服务端、Hooks | server-core |
| 服务端分发 | 请求分发到 handler、本地调用、服务端中间件 | server-core |
| 流式 | Stream/StreamX 抽象、双向流接入 | streaming |
| 收发管道 | TransHandler/TransPipeline、消息抽象 | remote-transport |
| 编解码 | Thrift/PB payload、TTHeader 消息、压缩、零拷贝 | remote-transport |
| 连接管理 | 连接池、拨号、远程会话 | remote-transport |
| 传输元信息 | 透传、自定义 meta handler | remote-transport |
| 传输实现 | netpoll / gonet / nphttp2(gRPC) / ttstream / 检测 | transport-implementations |
| 协议与泛化 | bthrift 线格式、泛化 codec、IDL 描述符 | protocol-generic |
| 服务治理 | 注册发现、负载均衡、重试、熔断降级、限流 ACL | governance |
| 可观测 | 统计/rpcinfo/事件、日志/logid/错误码、诊断剖析 | observability |
| 高级能力 | xDS 动态配置、代码生成工具、通用工具/预热/协程池 | advanced-tool |

## 3. 解决的问题

| 用户痛点 | Kitex 解法 |
|---|---|
| 自研 RPC 重复造轮子、性能参差 | 集成 netpoll 高性能网络库 + 零拷贝 bytebuf + 编解码流水线 |
| 多协议/多传输并存 | 消息协议（Thrift/Kitex PB/gRPC）与传输（TTHeader/HTTP2）解耦可换 |
| 治理能力散落、难定制 | 治理统一收敛为 endpoint 中间件，可插拔注入 |
| 服务发现/负载均衡需自研 | 提供 registry/discovery/loadbalance 接口与默认实现 |
| 线上故障难定位 | 内置 stats/klog/rpcinfo/logid/diagnosis 全链路观测 |
| 手写样板代码多 | kitex 代码生成工具从 IDL 生成 stub |

## 4. 系统边界

- **上边界（用户接入）**：业务代码通过 IDL 生成 stub 或泛化调用接入；`client/`、`server/` 是公开 API。
- **下边界（基础设施）**：网络 IO 下沉到外部 netpoll 库；序列化依赖 frugal/fastpb/prutal；代码生成依赖 thriftgo。
- **内边界（本仓库 vs 扩展）**：`internal/` 为隔离实现（client/server option 真正结构体、stream 内部状态机），公开包 re-export 自 internal；治理与传输均为接口，第三方可替换实现。
- **侧边界**：不内置注册中心后端实现、不内置监控/追踪后端，仅提供接入点。
- **不做什么**：不做服务编排/调度平台、不做消息队列、不做业务代码；HTTP2/gRPC 帧状态机为 vendored 移植实现。

## 5. 系统架构图说明

![系统架构图](system-architecture.html)

主路径自左向右：**业务代码/IDL stub → client 运行时 → 治理中间件链 → 收发管道与编解码 → 传输实现 → netpoll/内核 → 对端 Kitex 服务**。服务端对称：netpoll 接收事件 → 收发管道解码 → server invoke 分发到业务 handler。可观测埋点与注册中心/配置后端为横切关注点，贯穿 client/server 与治理层。

## 6. 核心时序图说明

![PingPong RPC 时序](system-pingpong-sequence.html)

一次 PingPong RPC 的完整往返：业务调用方发起 `Call` → client 编织重试/熔断/超时中间件 → 收发管道编码请求 → 写入 netpoll 连接 → TCP 帧到达对端 → 服务端解码并分发到 handler → 响应沿原路返回并经统计埋点 → 最终把 resp/err 交回调用方。

## 7. 系统数据流图说明

![编解码数据流](system-dataflow.html)

请求从业务对象出发：payload 编解码（Thrift/PB 序列化）→ 消息编解码（包 TTHeader/gRPC 协议头）→ 压缩 + bytebuf 零拷贝 → 连接池写出 TCP 帧。响应解码沿对称路径反向进行。

## 8. 质量与覆盖说明

- 共 **9 个域、29 个叶子**，每叶子 1 份设计文档级 MD + 至少 1 张渲染成功的 archify 图。
- 系统级三张图：系统架构图（standard）、PingPong 时序图（showcase）、编解码数据流图（showcase）。
- 语言适配口径：主语言 Go，重点覆盖并发模型 / goroutine 边界 / context 超时取消传播 / internal 边界（见各叶子 MD 第 9 节）。
- 外部组件（netpoll、thriftgo、frugal、注册中心/监控后端、对端服务）均在叶子 MD 第 7 节标注"不在本仓库源码内"。
