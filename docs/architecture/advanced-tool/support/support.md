# 通用工具与基础组件（support）

> 本文是 `advanced-tool` 域下的叶子子系统文档。域级总览见 `../advanced-tool.md`，本文只展开 `pkg/utils/`、`pkg/warmup/`、`pkg/mem/`、`pkg/gofunc/`、`internal/utils/`、`internal/configutil/` 这些被全框架复用的基础工具包。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 零拷贝转换 | `byte2str` 字节串零拷贝 | `pkg/utils/byte2str.go` |
| 计数器 | `counter.go` | `pkg/utils/counter.go` |
| 错误链 | `err_chain.go` | `pkg/utils/err_chain.go` |
| fastthrift | 高性能 thrift 编解码辅助 | `pkg/utils/fastthrift/` |
| sonic JSON | json_sonic 加速 | `pkg/utils/json_sonic.go` |
| context 映射 | `contextmap` | `pkg/utils/contextmap/` |
| 带恢复的 Go | `gofunc.RecoverGoFuncWithInfo` | `pkg/gofunc/go.go:48` |
| panic 处理器 | `SetPanicHandler` | `go.go:76` |
| 预热 | `warmup` + `pool_helper` | `pkg/warmup/warmup.go`、`pool_helper.go` |
| 内存 span | `mem.span` | `pkg/mem/span.go` |
| 安全 mcache | `internal/utils/safemcache` | `internal/utils/safemcache/` |
| 配置工具 | `internal/configutil`（config/once） | `internal/configutil/config.go`、`once.go` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `gofunc.GoTask`/`RecoverGoFuncWithInfo` | `go.go:31,48` | 统一 go 协程入口，带 panic 恢复 |
| `gofunc.Info` | `go.go:91` | 协程任务的远程服务/地址上下文 |
| `warmup` | `warmup.go` | 连接池等预热 |
| `mem.Span` | `span.go` | 内存块抽象 |

## 3. 关键调用链

1. **启动协程**：框架各处用 `gofunc.RecoverGoFuncWithInfo(ctx, task, info)`（`go.go:48`）替代裸 `go`，任务内 panic 被 recover 并经 `SetPanicHandler`（`go.go:76`）上报，不导致进程崩溃。
2. **零拷贝**：热路径用 `byte2str`（`byte2str.go`）避免 string 拷贝。
3. **预热**：`warmup` 在启动期预热连接池/缓冲，减少首次请求延迟。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `SetPanicHandler` | 自定义 panic 上报 | `go.go:76` |
| `gofunc` 全局默认 | init 注册默认 GoFunc | `go.go:36` |

## 5. 错误与重试语义

- panic 在 gofunc 内被 recover，不扩散；由 panic handler 决定上报。
- utils 纯函数无错误路径。

## 6. 并发细节

- **gofunc**：每个任务起独立 goroutine，自带 recover；是框架统一的并发出口。
- **safemcache**：内部安全 mcache，并发访问封装。
- **configutil/once**：sync.Once 式懒初始化。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/utils/`、`pkg/warmup/`、`pkg/mem/`、`pkg/gofunc/`、`internal/utils/`、`internal/configutil/`。

**Out-of-Scope（不在本仓库源码内）**
- sonic（bytedance/sonic）等第三方库是外部依赖。
- 这些包被全框架使用，具体业务使用点散见各叶子。

## 8. 与相邻子系统交互

- 上游 → 本叶子：全框架组件调用 gofunc/utils 等基础工具。
- 本叶子 → 下游：依赖标准库与少量外部库（sonic）。
- 流向：上层组件 → gofunc 起协程 / utils 工具函数 → 执行。

## 9. 语言专项适配口径

- **并发模型**：gofunc 是 Kitex 统一 goroutine 出口（带 recover 防护）；safemcache 封装并发安全。
- **控制器模式**：非 Reconcile。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界**：`internal/utils`、`internal/configutil` 是 internal 边界，仅本仓库内部使用，外部不可 import。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 通用工具架构图 | `support-architecture.html` | architecture | showcase |

- 本叶子不补 sequence/dataflow：工具包是被调用方，无自有管道。
- JSON IR 源文件：`json/support-architecture.json`。
