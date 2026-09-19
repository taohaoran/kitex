# Kitex 系统架构文档

> 基于 CloudWeGo Kitex 源码（`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`，约 756 个非测试 Go 文件 / 约 8.5 万行）深度分析产出，覆盖系统级、**9 个域、29 个叶子**子系统的功能、问题域、系统边界、架构图、时序图与数据流图。所有图表由 archify 渲染为自包含交互式 HTML。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 功能总览、解决的问题、系统边界、核心代码映射 | [系统架构图](system-architecture.html) · [PingPong 时序图](system-pingpong-sequence.html) · [编解码数据流图](system-dataflow.html) |

### client-core/（客户端运行时）

| 子系统 | 文档 | 架构图 | 时序图 |
|--------|------|--------|--------|
| 域总览 | [client-core.md](client-core/client-core.md) | — | — |
| client-entry | [client-entry.md](client-core/client-entry/client-entry.md) | [架构图](client-core/client-entry/client-entry-architecture.html) | [时序图](client-core/client-entry/client-entry-sequence.html) |
| client-middleware-endpoint | [client-middleware-endpoint.md](client-core/client-middleware-endpoint/client-middleware-endpoint.md) | [架构图](client-core/client-middleware-endpoint/client-middleware-endpoint-architecture.html) | [时序图](client-core/client-middleware-endpoint/client-middleware-endpoint-sequence.html) |

### server-core/（服务端运行时）

| 子系统 | 文档 | 架构图 | 时序图 |
|--------|------|--------|--------|
| 域总览 | [server-core.md](server-core/server-core.md) | — | — |
| server-entry | [server-entry.md](server-core/server-entry/server-entry.md) | [架构图](server-core/server-entry/server-entry-architecture.html) | [时序图](server-core/server-entry/server-entry-sequence.html) |
| server-invoke | [server-invoke.md](server-core/server-invoke/server-invoke.md) | [架构图](server-core/server-invoke/server-invoke-architecture.html) | [时序图](server-core/server-invoke/server-invoke-sequence.html) |

### streaming/（流式）

| 子系统 | 文档 | 架构图 | 时序图 |
|--------|------|--------|--------|
| 域总览 | [streaming.md](streaming/streaming.md) | — | — |
| streaming-core | [streaming-core.md](streaming/streaming-core/streaming-core.md) | [架构图](streaming/streaming-core/streaming-core-architecture.html) | [时序图](streaming/streaming-core/streaming-core-sequence.html) |
| stream-client-server | [stream-client-server.md](streaming/stream-client-server/stream-client-server.md) | [架构图](streaming/stream-client-server/stream-client-server-architecture.html) | [时序图](streaming/stream-client-server/stream-client-server-sequence.html) |

### remote-transport/（远程收发与编解码）

| 子系统 | 文档 | 架构图 | 时序/数据流/生命周期 |
|--------|------|--------|----------------------|
| 域总览 | [remote-transport.md](remote-transport/remote-transport.md) | — | — |
| trans-handler-pipeline | [trans-handler-pipeline.md](remote-transport/trans-handler-pipeline/trans-handler-pipeline.md) | [架构图](remote-transport/trans-handler-pipeline/trans-handler-pipeline-architecture.html) | [时序图](remote-transport/trans-handler-pipeline/trans-handler-pipeline-sequence.html) |
| codec-payload | [codec-payload.md](remote-transport/codec-payload/codec-payload.md) | [架构图](remote-transport/codec-payload/codec-payload-architecture.html) | [数据流图](remote-transport/codec-payload/codec-payload-dataflow.html) |
| connpool-dialer | [connpool-dialer.md](remote-transport/connpool-dialer/connpool-dialer.md) | [架构图](remote-transport/connpool-dialer/connpool-dialer-architecture.html) | [数据流图](remote-transport/connpool-dialer/connpool-dialer-dataflow.html) · [生命周期](remote-transport/connpool-dialer/connpool-dialer-lifecycle.html) |
| transmeta | [transmeta.md](remote-transport/transmeta/transmeta.md) | [架构图](remote-transport/transmeta/transmeta-architecture.html) | [数据流图](remote-transport/transmeta/transmeta-dataflow.html) |

### transport-implementations/（传输实现）

| 子系统 | 文档 | 架构图 | 时序图 |
|--------|------|--------|--------|
| 域总览 | [transport-implementations.md](transport-implementations/transport-implementations.md) | — | — |
| netpoll-trans | [netpoll-trans.md](transport-implementations/netpoll-trans/netpoll-trans.md) | [架构图](transport-implementations/netpoll-trans/netpoll-trans-architecture.html) | [时序图](transport-implementations/netpoll-trans/netpoll-trans-sequence.html) |
| gonet-trans | [gonet-trans.md](transport-implementations/gonet-trans/gonet-trans.md) | [架构图](transport-implementations/gonet-trans/gonet-trans-architecture.html) | [时序图](transport-implementations/gonet-trans/gonet-trans-sequence.html) |
| nphttp2-grpc | [nphttp2-grpc.md](transport-implementations/nphttp2-grpc/nphttp2-grpc.md) | [架构图](transport-implementations/nphttp2-grpc/nphttp2-grpc-architecture.html) | [时序图](transport-implementations/nphttp2-grpc/nphttp2-grpc-sequence.html) |
| ttstream-mux | [ttstream-mux.md](transport-implementations/ttstream-mux/ttstream-mux.md) | [架构图](transport-implementations/ttstream-mux/ttstream-mux-architecture.html) | [时序图](transport-implementations/ttstream-mux/ttstream-mux-sequence.html) |
| detection-invoke | [detection-invoke.md](transport-implementations/detection-invoke/detection-invoke.md) | [架构图](transport-implementations/detection-invoke/detection-invoke-architecture.html) | [时序图](transport-implementations/detection-invoke/detection-invoke-sequence.html) |

### protocol-generic/（协议与泛化）

| 子系统 | 文档 | 架构图 | 数据流图 |
|--------|------|--------|----------|
| 域总览 | [protocol-generic.md](protocol-generic/protocol-generic.md) | — | — |
| bthrift-protocol | [bthrift-protocol.md](protocol-generic/bthrift-protocol/bthrift-protocol.md) | [架构图](protocol-generic/bthrift-protocol/bthrift-protocol-architecture.html) | — |
| generic-codec | [generic-codec.md](protocol-generic/generic-codec/generic-codec.md) | [架构图](protocol-generic/generic-codec/generic-codec-architecture.html) | [数据流图](protocol-generic/generic-codec/generic-codec-dataflow.html) |
| generic-descriptor | [generic-descriptor.md](protocol-generic/generic-descriptor/generic-descriptor.md) | [架构图](protocol-generic/generic-descriptor/generic-descriptor-architecture.html) | [数据流图](protocol-generic/generic-descriptor/generic-descriptor-dataflow.html) |

### governance/（服务治理）

| 子系统 | 文档 | 架构图 | 时序/生命周期 |
|--------|------|--------|----------------|
| 域总览 | [governance.md](governance/governance.md) | — | — |
| registry-discovery | [registry-discovery.md](governance/registry-discovery/registry-discovery.md) | [架构图](governance/registry-discovery/registry-discovery-architecture.html) | [时序图](governance/registry-discovery/registry-discovery-sequence.html) |
| loadbalance-endpoint | [loadbalance-endpoint.md](governance/loadbalance-endpoint/loadbalance-endpoint.md) | [架构图](governance/loadbalance-endpoint/loadbalance-endpoint-architecture.html) | [时序图](governance/loadbalance-endpoint/loadbalance-endpoint-sequence.html) |
| retry | [retry.md](governance/retry/retry.md) | [架构图](governance/retry/retry-architecture.html) | [生命周期](governance/retry/retry-lifecycle.html) |
| circuitbreak-fallback | [circuitbreak-fallback.md](governance/circuitbreak-fallback/circuitbreak-fallback.md) | [架构图](governance/circuitbreak-fallback/circuitbreak-fallback-architecture.html) | [生命周期](governance/circuitbreak-fallback/circuitbreak-fallback-lifecycle.html) |
| limit-limiter | [limit-limiter.md](governance/limit-limiter/limit-limiter.md) | [架构图](governance/limit-limiter/limit-limiter-architecture.html) | [时序图](governance/limit-limiter/limit-limiter-sequence.html) |

### observability/（可观测）

| 子系统 | 文档 | 架构图 | 数据流图 |
|--------|------|--------|----------|
| 域总览 | [observability.md](observability/observability.md) | — | — |
| stats-rpcinfo | [stats-rpcinfo.md](observability/stats-rpcinfo/stats-rpcinfo.md) | [架构图](observability/stats-rpcinfo/stats-rpcinfo-architecture.html) | [数据流图](observability/stats-rpcinfo/stats-rpcinfo-dataflow.html) |
| klog-logid-kerrors | [klog-logid-kerrors.md](observability/klog-logid-kerrors/klog-logid-kerrors.md) | [架构图](observability/klog-logid-kerrors/klog-logid-kerrors-architecture.html) | — |
| diagnosis-profiler | [diagnosis-profiler.md](observability/diagnosis-profiler/diagnosis-profiler.md) | [架构图](observability/diagnosis-profiler/diagnosis-profiler-architecture.html) | — |

### advanced-tool/（高级能力与工具）

| 子系统 | 文档 | 架构图 | 数据流图 |
|--------|------|--------|----------|
| 域总览 | [advanced-tool.md](advanced-tool/advanced-tool.md) | — | — |
| xds | [xds.md](advanced-tool/xds/xds.md) | [架构图](advanced-tool/xds/xds-architecture.html) | — |
| codegen | [codegen.md](advanced-tool/codegen/codegen.md) | [架构图](advanced-tool/codegen/codegen-architecture.html) | [数据流图](advanced-tool/codegen/codegen-dataflow.html) |
| support | [support.md](advanced-tool/support/support.md) | [架构图](advanced-tool/support/support-architecture.html) | — |

## 产出统计

| 层级 | MD | HTML 图 | JSON IR |
|------|----|---------|---------|
| 系统级 | 2（README + system-overview） | 3 | 3 |
| 9 个域总览 | 9 | 0 | 0 |
| 29 个叶子 | 29 | 54 | 54 |
| **合计** | **40** | **57** | **57** |

## 覆盖范围与说明

- **叶子清单**：client-core(2)、server-core(2)、streaming(2)、remote-transport(4)、transport-implementations(5)、protocol-generic(3)、governance(5)、observability(3)、advanced-tool(3)，共 29 个。
- **质量档位**：多数叶子图为 showcase；少量因 archify 布局校验（连线穿节点 / 双向回边 / 标签间距）按流程降为 standard，已在对应叶子 MD 第 10 节如实披露；所有图 render 退出码均为 0。
- **语言适配口径**：主语言 Go，各叶子 MD 第 9 节覆盖并发模型 / goroutine 边界 / context 超时取消传播 / internal 边界。
- **外部依赖标注**：netpoll、thriftgo、frugal/fastpb/prutal、注册中心与监控后端、对端服务，均在叶子 MD 第 7 节标注"不在本仓库源码内"。
- **已知边界**：nphttp2-grpc 的 HTTP2/gRPC 帧状态机为 vendored 移植实现，未逐行展开；流状态机位于传输层，streaming 域未单列 lifecycle（已在 MD 说明理由）。
