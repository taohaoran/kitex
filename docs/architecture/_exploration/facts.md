# Kitex 项目探查事实（facts.md）

> 一次性只读探查产物。所有分片/叶子直接读本文件，不要再各自重扫仓库。

## 1. 身份

- 模块：`github.com/cloudwego/kitex`（go.mod）
- Go 版本：go 1.20
- 语言：**主语言 Go**（Go RPC 框架，无 cgo 核心；netpoll 为外部依赖）
- git commit：`4fffa48 chore: bump localsession to v0.2.2 and runtimex to v0.1.2 (#1998)`
- 定位：CloudWeGo 开源的**高性能、强可扩展 Go RPC 框架**（字节跳动开源，CNCF 景观项目）。Apache-2.0。
- 关键外部依赖：netpoll（网络库）、frugal/fastpb/prutal（序列化）、dynamicgo（动态 Thrift）localsession、thriftgo（代码生成）、configmanager、gopkg。

## 2. 规模

- 非 vendor 非测试 Go 文件约 756 个；非测试代码约 **85,004 行**。
- 各顶层目录 go 文件数：client 38、server 32、pkg 556、internal 72、transport 1（仅 keys.go）、tool 56。
- 无 `cmd/` 多二进制入口；唯一二进制入口是代码生成工具 `tool/cmd/kitex/main.go`（`package main`），运行时库本身是被 import 的库。
- 最大非测试文件：
  - internal/mocks/thrift/k-testservice.go (1487) — mock，非业务
  - pkg/remote/trans/nphttp2/grpc/http2_client.go (1443)、http2_server.go (1211)、transport.go (1045)、controlbuf.go (1006) — gRPC/HTTP2 移植实现
  - tool/internal_pkg/pluginmode/thriftgo/struct_tpl.go (1100) — 代码生成模板
  - client/client.go (854)、pkg/generic/thrift/parse.go (810)、write.go (796)、tool/internal_pkg/generator/generator.go (683)、pkg/generic/generic.go (679)

## 3. 结构（一层）

```
kitex/
├── client/        # RPC 客户端：client.go(854行)、option*.go、callopt/、middlewares.go、
│                  #   context*.go、rpctimeout*.go、stream*.go、service_inline.go、genericclient/、streamclient/
├── server/        # RPC 服务端：server.go、service.go、option*.go、invoke.go、local_caller.go、
│                  #   hooks.go、middlewares.go、stream*.go、service_inline.go、genericserver/
├── pkg/           # 库主体（556 文件）
│   ├── remote/    # 远程通信核心：trans_handler.go / trans_pipeline.go / trans_server.go / message.go /
│   │              #   codec.go / payload_codec.go / connpool*/ / dialer.go / bytebuf.go / transmeta/ / bound/
│   │   └── trans/  # 传输实现：netpoll/ gonet/ nphttp2/(grpc/) ttstream/ netpollmux/ detection/ invoke/ internal/
│   ├── protocol/   # 仅 bthrift/（Thrift 协议线格式实现）
│   ├── generic/    # 泛化调用 codec：binary/json/http/map thrift 与 pb；descriptor/ thrift/ proto/ *idl_provider*.go
│   ├── streaming/  # 流抽象 streaming.go streamx.go context.go timeout.go
│   ├── registry/ discovery/ loadbalance/ endpoint/   # 注册发现 + 负载 + endpoint 中间件
│   ├── retry/ circuitbreak/ fallback/ limit/ limiter/ acl/   # 容错与限流
│   ├── stats/ rpcinfo/ event/ klog/ logid/ kerrors/ exception/ transmeta/ diagnosis/ profiler/  # 可观测
│   ├── xds/ proxy/ http/ warmup/ connpool/ mem/ gofunc/ utils/ consts/ serviceinfo/ rpctimeout/
├── internal/      # 内部实现：client/ server/ stream/ generic/ configutil/ utils/ mocks/ test/ reusable.go
├── tool/          # 代码生成：cmd/kitex(main.go, args/, sdk/, utils/, versions/) 与 internal_pkg/(generator/ pluginmode/ tpl/ prutal/ util/ log/)
└── transport/     # 仅 keys.go（传输层 key 常量）
```

## 4. 功能清单（README Feature List 摘录）

1. 高性能：集成 Netpoll，相比 go net 有显著性能优势。
2. 强扩展：大量接口+默认实现，可自定义注入。
3. 多消息协议：Thrift、Kitex Protobuf、gRPC；可扩展自定义协议。
4. 多传输协议：TTHeader（配 Thrift/Kitex PB）、HTTP2（配 gRPC）。
5. 多消息类型：PingPong、One-way（仅 Thrift）、Bidirectional Streaming。
6. 服务治理：注册/发现、负载均衡、熔断、限流、重试、监控、tracing、日志、诊断。
7. 代码生成：Thrift / Protobuf / scaffold 代码生成。

## 5. 约束

- **项目根无 docs/ 目录** → 按输出根规则，输出根为 `<repo>/docs/architecture/`（本文件所在目录）。
- **所有输出物必须简体中文**（硬性）：MD/README/HTML 图作者文案/JSON IR 作者字段一律简体；代码标识符、路径、flag 名保持原文。
- 文件名/目录名一律英文短横线（-）。
- AGENTS.md 无。License Apache-2.0。

## 6. 主语言判定与语言专项

- 主语言 **Go**（K8s 风格不适用；这是网络 RPC 框架，重点是并发模型/goroutine 边界/连接池/编解码流水线/context 超时取消传播）。
- 图型侧重：编解码/事件管道 → dataflow；RPC 调用链/消息交互 → sequence；并发通信（连接池、流）→ sequence；组件分层 → architecture；连接/会话生命周期 → lifecycle。

## 7. archify 工具链（已就绪，分片直接用）

- `ARCHIFY_CLI=/tmp/archify-upstream/archify/bin/archify.mjs`（Node v22，已验证 --help 正常）。
- 渲染推荐脚本（自动 showcase→standard 回退并报告档位）：
  `bash <skill_root>/scripts/render-diagram.sh <type> <x.json> <out.html>`
  skill_root = `/Users/thr/Library/Application Support/DoubaoWork/Default/.doubaowork/agent_mode/workspace/.user_skills/archify-codebase-analysis`
- 链接校验：`python3 <skill_root>/scripts/check-links.py <输出根>`
- 出图速查：`<skill_root>/references/archify-usage-guide.md`（五图最小模板、报错修复顺序）。
- 叶子 MD 模板：`<skill_root>/references/leaf-template.md`（10 小节必用）。
- 输出结构模板：`<skill_root>/references/output-structure.md`。

## 8. 依赖方向要点（初判）

- 外部 API 包：`client/`、`server/`、`pkg/`（公开）；`internal/` 为隔离实现，client/server 内部 option 均 re-export 自 internal。
- 调用方向：用户业务 → client 包 → pkg/remote（trans_handler/pipeline）→ pkg/remote/trans（具体传输 netpoll/gonet/nphttp2）→ 对端；服务端对称：trans 接收 → remote → server.invoke → 业务 handler。
- 治理中间件以 `endpoint.Middleware`（pkg/endpoint）形式编织进 client/server 调用链（retry/circuitbreak/limit/stats 等均为 endpoint 中间件或 trans 层钩子）。
