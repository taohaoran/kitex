# kitex 代码生成工具（codegen）

> 本文是 `advanced-tool` 域下的叶子子系统文档。域级总览见 `../advanced-tool.md`，本文只展开 `tool/cmd/kitex/`（唯一二进制入口）、`tool/internal_pkg/generator/`、`pluginmode/`、`tpl/`、`prutal/`，不展开运行时 RPC 框架本身。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 双模式入口 | main 按环境变量区分插件模式 / CLI 模式 | `tool/cmd/kitex/main.go:68-79` |
| 参数解析 | `args.ParseArgs` | `main.go:87` |
| 依赖版本校验 | `versions.DefaultCheckDependencyAndProcess` | `main.go:97` |
| IDL 拉取 | git@/http(s) 前缀的 include 自动 clone | `main.go:103-115` |
| protobuf 生成 | prutal（纯Go）或 protoc 插件二选一 | `main.go:117-127` |
| 生成器接口 | `Generator` + `Config` + Middleware 链 | `tool/internal_pkg/generator/generator.go:99,106,332` |
| 插件模式 | `thriftgo.Run` / `protoc.Run` | `pluginmode/thriftgo`、`pluginmode/protoc` |
| 纯Go protobuf 生成 | `prutal.NewPrutalGen` | `tool/internal_pkg/prutal/prutal.go` |
| 模板 | `tpl/` 模板集 + custom_template | `generator/template.go`、`custom_template.go` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `main` | `cmd/kitex/main.go:68` | 唯一二进制入口 |
| `generator.Generator` | `generator.go:99` | 代码生成器抽象 |
| `generator.Config` | `generator.go:106` | 生成配置（编译器路径、特性开关等） |
| `Middleware`/`HandleFunc` | `generator.go:332-335` | 模板渲染中间件链 |
| `prutal` | `prutal.go` | 纯 Go protobuf 代码生成 |

## 3. 关键调用链

1. **插件模式**：环境变量 `EnvPluginMode` 非空且无参数时，按 `thriftgo.PluginName`/`protoc.PluginName` 调对应 `Run()`（`main.go:72-77`）作为 IDL 编译器插件运行。
2. **CLI 模式**：`ParseArgs`（`main.go:87`）→ 依赖校验（`main.go:97`）→ 处理 git include（`main.go:103`）→ 若是 protobuf 走 prutal/protoc（`main.go:117`），否则走 thriftgo 生成。
3. **模板渲染链**：`NewGenerator(config, middlewares)`（`generator.go:322`）→ `chainMWs`（`generator.go:342`）把多个 Middleware 串成 HandleFunc → `GenerateMainPackage`（`generator.go:349`）渲染输出文件。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `Config.CompilerPath` | 指定 thriftgo/protoc 路径 | `generator.go:130` |
| `Config.NoFastAPI` | protobuf 不再生成 fast api | `main.go:119` |
| `NoDependencyCheck` | 跳过依赖版本校验 | `main.go:95` |
| `IsProtobuf/UseProtoc` | 选 IDL 类型与生成后端 | `main.go:117,120` |

## 5. 错误与重试语义

- 生成失败直接 `os.Exit(1/2)`，不重试。
- git clone 失败提示移除 `~/.kitex` 重试（`main.go:110`）。
- 依赖版本不兼容用专门退出码 `CompatibilityCheckExitCode`（`main.go:98`）。

## 6. 并发细节

- 这是 CLI 工具，单次进程跑完即退，无长驻 goroutine。
- 模板渲染是同步串行（Middleware 链）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `tool/cmd/kitex/`、`tool/internal_pkg/generator/`、`pluginmode/`、`tpl/`、`prutal/`。

**Out-of-Scope（不在本仓库源码内）**
- thriftgo（`github.com/cloudwego/thriftgo`）与 protoc 本体是外部编译器，不在本仓库；本工具以插件方式调用。
- 生成出的客户端/服务端骨架代码本身不在本仓库源码内（是产物）。

## 8. 与相邻子系统交互

- 上游：开发者命令行调用 `kitex` 工具。
- 下游：调用 thriftgo/protoc 插件、git、写生成文件到磁盘。
- 流向：参数 → 依赖校验 → IDL 解析 → 模板渲染 → 生成 Go 文件。

## 9. 语言专项适配口径

- **并发模型**：CLI 单次执行，无并发。
- **控制器模式**：非 Reconcile。
- **多二进制与部署边界**：这是项目唯一的独立二进制 `tool/cmd/kitex`（其余为库）；编译产物是 `kitex` 命令行工具。
- **internal 边界**：`tool/internal_pkg/` 是 internal 边界，仅被 `tool/cmd/kitex` 引用，外部不可 import。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 代码生成工具架构图 | `codegen-architecture.html` | architecture | showcase |
| 代码生成流水线 | `codegen-dataflow.html` | dataflow | showcase |

- 本叶子不补 sequence/lifecycle：生成是一次性流水线，已由数据流图表达。
- JSON IR 源文件：`json/codegen-architecture.json`、`json/codegen-dataflow.json`。
