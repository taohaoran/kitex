# 结构化日志 / 日志ID / 错误码（klog-logid-kerrors）

> 本文是 `observability` 域下的叶子子系统文档。域级总览见 `../observability.md`，本文只展开 `pkg/klog/`（结构化日志门面）、`pkg/logid/`（日志 ID）、`pkg/kerrors/`（错误码与业务错误）、`pkg/exception/`（已废弃），不展开 metrics 采集（见 `../stats-rpcinfo/stats-rpcinfo.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 日志门面 | `klog.Trace/Debug/Info/.../Fatal` 各级便捷函数 | `pkg/klog/default.go:57-87` |
| Logger 接口 | `FormatLogger/Logger/FullLogger` | `pkg/klog/log.go:26,37` |
| 日志替换 | `SetLogger(v FullLogger)` / `SetOutput` / `SetLevel` | `default.go:33,40,52` |
| 日志ID生成 | `DefaultLogIDGenerator(ctx)`，可 `SetLogIDGenerator` | `pkg/logid/logid.go:46,72` |
| 错误码分类 | `ErrInternalException/ErrOverlimit/ErrCircuitBreak/ErrServiceDiscovery` | `pkg/kerrors/kerrors.go:58-68` |
| 业务错误 | `BizStatusErrorIface` + `NewBizStatusError/FromBizStatusError` | `pkg/kerrors/bizerrors.go:66` |
| gRPC 业务错误 | `NewGRPCBizStatusError` + `GRPCStatusIface` | `bizerrors.go:80,74` |
| 流式错误 | streaming_errors.go | `pkg/kerrors/streaming_errors.go` |
| 异常（废弃） | `pkg/exception/` 已标记 deprecated | `pkg/exception/deprecated.go` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `klog.FullLogger` | `log.go` | 日志完整接口（含格式化与级别） |
| `DefaultLogger` | `default.go:45` | 当前全局 logger，可替换 |
| `DefaultLogIDGenerator` | `logid.go:72` | 默认 logid 生成函数 |
| `kerrors` 错误变量 | `kerrors.go:58+` | 预置分类错误码 |
| `BizStatusErrorIface` | `bizerrors.go:66` | 业务状态错误接口 |

## 3. 关键调用链

1. **日志输出**：业务/框架调 `klog.Error(...)`（`default.go:62`）→ 转发到当前 `DefaultLogger`；用户可 `SetLogger` 注入自定义 logger（如 zap/logrus 适配）。
2. **logid 生成**：RPC 开始经 `DefaultLogIDGenerator(ctx)`（`logid.go:72`）生成一次 logid 挂在 ctx，全程日志携带；`SetLogIDGenerator`（`logid.go:46`）可替换生成算法。
3. **错误分类**：框架内部错误用 `ErrXxx.WithCause(...)`（`kerrors.go:58-68`）构造，调用方用 `errors.Is` 判定类型；业务侧用 `NewBizStatusError(code, msg)`（`bizerrors.go`）返回业务码，对端 `FromBizStatusError` 还原。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `SetLevel` | 日志级别阈值 | `default.go:40` |
| `SetOutput` | 输出 writer（默认 stdout） | `default.go:33` |
| `SetLogIDGenerator` | 自定义 logid 生成 | `logid.go:46` |
| `SetLogIDExtra` | logid 附加串 | `logid.go:52` |

## 5. 错误与重试语义

- kerrors 错误是分类化的，不触发重试；重试/熔断据错误类型决策（见 retry/circuitbreak-fallback 叶子）。
- `BizStatusError` 是业务层错误，与框架错误分离；对端通过 `FromBizStatusError` 判定。
- 日志本身无失败重试。

## 6. 并发细节

- 全局 logger 指针可热替换（`SetLogger`），调用方读 `DefaultLogger`。
- logid 生成是无状态函数式，按 ctx 生成；`init()` 注册默认生成器（`logid.go:40`）。
- 无 goroutine/channel。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/klog/`、`pkg/logid/`、`pkg/kerrors/`、`pkg/exception/`。

**Out-of-Scope（不在本仓库源码内）**
- 具体日志后端实现（zap/logrus/文件）由用户注入，不在本仓库。
- 指标/追踪见 `../stats-rpcinfo/stats-rpcinfo.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：框架各包用 klog 打日志；错误用 kerrors 返回。
- 本叶子 → 下游：klog 转发到用户注入 logger；kerrors 错误向上传播给调用方。
- 流向：框架运行 → klog 记录（含 logid）→ 用户 logger 输出；错误 → kerrors 分类 → 调用方 errors.Is 判定。

## 9. 语言专项适配口径

- **并发模型**：全局 logger 可热替换；logid 函数式无状态。
- **控制器模式**：非 Reconcile。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：`pkg/klog`、`pkg/kerrors` 是被全仓库广泛依赖的基础包，不依赖业务层，方向纯净。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 日志/错误码架构图 | `klog-logid-kerrors-architecture.html` | architecture | showcase |

- 本叶子不补 sequence/dataflow/lifecycle：日志/错误码是同步调用与常量定义，无管道或状态迁移。
- JSON IR 源文件：`json/klog-logid-kerrors-architecture.json`。
