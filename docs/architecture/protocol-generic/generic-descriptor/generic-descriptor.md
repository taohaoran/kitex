# IDL 描述符解析与 Provider（generic-descriptor）

> 本文是 `protocol-generic` 域下的叶子子系统文档。域级总览见 `../protocol-generic.md`，本文只展开 `pkg/generic/descriptor/` 的 IDL 数据模型与 HTTP 路由，以及 `pkg/generic/*idl_provider*.go` 的描述符 Provider 实现；不重复展开各 codec 如何消费描述符（见 `../generic-codec/generic-codec.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 服务描述符数据模型 | `ServiceDescriptor/FunctionDescriptor/StructDescriptor/FieldDescriptor/TypeDescriptor` 五层 IDL 描述结构 | `pkg/generic/descriptor/descriptor.go:104,93,74,32,64` |
| 方法查找 | `ServiceDescriptor.LookupFunctionByMethod` 按方法名查 `FunctionDescriptor` | `descriptor.go:113` |
| 必填字段校验 | `StructDescriptor.CheckRequired` 读写结束后校验 required 字段是否齐全 | `descriptor.go:83` |
| 字段名别名 | `FieldDescriptor.FieldName()` 按 go tag 别名开关决定返回 Alias 还是 Name | `descriptor.go:51` |
| Thrift 类型枚举 | `Type`（STOP/BOOL/STRING/STRUCT/MAP...）与 `String()`、废弃的 `ToThriftTType` | `descriptor/type.go:26,70` |
| HTTP 请求/响应抽象 | `HTTPRequest/HTTPResponse`，GetHeader/GetQuery/GetBody/GetParam 等访问器 | `descriptor/http.go:44,161` |
| HTTP 字段映射 | `HTTPMapping` 描述 thrift 字段到 HTTP 位置（path/query/header/body）的映射 | `descriptor/http_mapping.go`、`descriptor.go:41` |
| 值映射 | `ValueMapping` 描述枚举/常量到 HTTP 取值的映射 | `descriptor/value_mapping.go`、`descriptor.go:42` |
| 注解解析 | `annotation.go` 解析 IDL 注解（thrift annotation） | `descriptor/annotation.go` |
| HTTP 路由基数树 | `Router` 接口 + `router` 实现，`Handle(Route)` 注册、`Lookup(*HTTPRequest)` 按路径匹配到 `FunctionDescriptor`；底层 radix tree | `descriptor/router.go:25,55,97`、`descriptor/tree.go` |
| Thrift 文件 Provider | `NewThriftFileProvider[WithDynamicGo]` 从磁盘路径解析 thrift IDL | `pkg/generic/thriftidl_provider.go:50,70` |
| Thrift 内存内容 Provider | `NewThriftContentProvider[WithDynamicGo]` / `...WithAbsIncludePath` 从字符串内容解析 | `thriftidl_provider.go:166,192` |
| Thrift 描述符构造 | `newServiceDescriptorFromPath`：`parser.ParseFile` → `thrift.Parse(tree, mode, opts)` | `thriftidl_provider.go:108,120` |
| Protobuf Provider | `NewPbContentProvider` / `NewPbFileProviderWithDynamicGo`，`UpdateIDL` 热更新 | `pkg/generic/pbidl_provider.go:45,115,147` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `DescriptorProvider` | `pkg/generic/descriptor_provider.go:26` | `Provide() <-chan *ServiceDescriptor` + `Closer`；IDL 描述符的统一推送源 |
| `ProviderOption` | `descriptor_provider.go:37` | `DynamicGoEnabled` 与 `DynamicGoOptions`，告知 codec 是否启用 dynamicgo 加速 |
| `thriftFileProvider` / `ThriftContentProvider` | `thriftidl_provider.go:43,148` | thrift IDL Provider 具体实现；内部 `svcs chan *descriptor.ServiceDescriptor`（cap 1 缓冲） |
| `PbContentProvider` / `PbFileProviderWithDynamicGo` | `pbidl_provider.go:30,35` | protobuf IDL Provider，分别产出 `proto.ServiceDescriptor` 与 dynamicgo `*dproto.ServiceDescriptor` |
| `Router` | `router.go:25` | HTTP 路由抽象：`Handle(Route)` 注册、`Lookup(*HTTPRequest)` 查找 |
| `ServiceDescriptor` | `descriptor.go:104` | 顶层：服务名、Functions map、Router、`DynamicGoDsc *dthrift.ServiceDescriptor`、是否合并多服务 |
| `FunctionDescriptor` | `descriptor.go:93` | 方法：Oneway、Request/Response 类型、StreamingMode、是否无 wrapping |
| `FieldDescriptor` | `descriptor.go:32` | 字段：ID/Required/Optional/DefaultValue/Type/HTTPMapping/ValueMapping/GoTagOpt |

## 3. 关键调用链

1. **从磁盘文件构建 Thrift Provider**：`NewThriftFileProviderWithOption(path, opts, includeDirs...)`（`thriftidl_provider.go:54`）建一个 `svcs chan *descriptor.ServiceDescriptor`（cap=1，`thriftidl_provider.go:58`），随后调 `newServiceDescriptorFromPath`（`thriftidl_provider.go:108`）：先 `parser.ParseFile(path, includeDirs, true)`（`thriftidl_provider.go:109`）得到 AST tree，再 `thrift.Parse(tree, parseMode, parseOpts...)`（`thriftidl_provider.go:120`）由 dynamicgo 产出 `*descriptor.ServiceDescriptor`，最后 `p.svcs <- svc` 推入缓冲 channel。
2. **Provider 推送与消费**：消费方（泛化 codec）调 `Provide()` 拿只读 channel（`thriftidl_provider.go:131`）；首次同步阻塞取一个，之后阻塞等后续更新。`UpdateIDL`/文件重新解析会再次 `p.svcs <- svc` 推送新描述符（pb 侧见 `pbidl_provider.go:61,147`）。`Close()` 用 `sync.Once` 关闭发送 channel（`thriftidl_provider.go:136`），消费方 `!ok` 退出。
3. **HTTP 泛化路由查找**：服务描述符建好后 `Router.Handle(Route)`（`router.go:55`）把每个方法的 HTTP 路由注册进基数树（`tree.go`）；收到 HTTP 请求时 `router.Lookup(req)`（`router.go:97`）沿 radix tree 匹配 path/query，返回 `*FunctionDescriptor`，再由 httpThriftCodec 做字段映射。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| 环境变量 `KITEX_GENERIC_GOTAG_ALIAS_DISABLED=True` | 全局禁用 go tag 别名；否则字段名按 `GoTagOpt` 决定 | `descriptor.go:29,53` |
| `ThriftIDLProviderOption`（go tag / serviceName / dynamicgo 选项） | 经 `thriftidl_provider_option.go` 应用；`WithIDLServiceName` 指定服务名 | `thriftidl_provider.go:117` |
| `ProviderOption.DynamicGoEnabled` | false 走原生解析，true 走 dynamicgo（更快） | `descriptor_provider.go:39` |
| `parseMode`（thrift.ParseMode） | 由 provider option 推导，控制 thrift 解析模式 | `thriftidl_provider.go:169` |

## 5. 错误与重试语义

- IDL 解析失败（`parser.ParseFile` / `thrift.Parse` 报错）时 Provider 构造函数直接返回 error（`thriftidl_provider.go:110,121`），不进入运行时。
- `LookupFunctionByMethod` 方法不存在返回 `"missing method: %s in service: %s"`（`descriptor.go:116`）。
- `CheckRequired` 发现缺必填字段返回 `"required field (%d/%s) missing"`（`descriptor.go:86`），由 codec 在读/写结束后调用。
- Provider 不重试解析；`UpdateIDL` 失败返回 error，旧描述符仍在 channel 中继续可用。
- 文件 watch 热更新仅为注释中的 TODO（`thriftidl_provider.go:127`），当前不自动监听文件变化，需用户主动 `UpdateIDL`。

## 6. 并发细节

- **channel 模型**：每个 Provider 持有一个 cap=1 的 buffered channel（`thriftidl_provider.go:58`），单写者（Provider 自身）、单读者（消费 codec 的 `update()` 协程）；缓冲 1 保证首次推送不阻塞。
- **goroutine 启停**：Provider 构造时同步解析并推首个描述符，不额外起 goroutine；消费方 codec 起 `go c.update()` 读 channel（见 generic-codec 叶子）。`Close()` 经 `sync.Once` 幂等关闭 channel（`thriftidl_provider.go:137`），避免重复 close panic。
- **基数树并发**：`router` 用 `getParams/putParams` 复用 `Params` 对象池（`router.go:43,49`），`Handle` 只在建描述符时调用一次，`Lookup` 是只读并发安全的查找。
- context：dynamicgo 解析在构造期同步进行，不携带请求 context；`NewPbFileProviderWithDynamicGo` 接收 `ctx` 用于解析生命周期（`pbidl_provider.go:115`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/generic/descriptor/`：IDL 描述数据模型、HTTP 抽象与映射、注解、基数树路由。
- `pkg/generic/thriftidl_provider.go`、`thriftidl_provider_option.go`、`pbidl_provider.go`、`descriptor_provider.go`、`pb_descriptor_provider.go`：各类 IDL Provider。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/cloudwego/dynamicgo/thrift`（`parser.ParseFile`、`thrift.Parse`、`dthrift.ServiceDescriptor`）与 `dynamicgo/proto`：真正的 IDL AST 解析与描述符生成，本包只做薄封装与模型转换。
- `github.com/cloudwego/gopkg/protocol/thrift`：Thrift 类型常量来源。
- 各 codec 如何用描述符做动态编解码见 `../generic-codec/generic-codec.md`；HTTP 服务端如何把请求接到 router 属 server 分片。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户调用 `generic.NewThriftFileProvider/NewThriftContentProvider/NewPbContentProvider` 等工厂，传入 IDL 路径或内容。
- 本叶子 → 下游：Provider 经 `Provide()` channel 把 `*descriptor.ServiceDescriptor` 推给泛化 codec（generic-codec 叶子）；codec 用 `FunctionDescriptor/FieldDescriptor` 驱动 `thrift.StructReaderWriter`；HTTP 泛化用 `Router.Lookup` 选方法。
- 流向：IDL 文本 → dynamicgo 解析 → ServiceDescriptor → channel → codec 原子切换读写器 → 在线请求。

## 9. 语言专项适配口径

- **并发模型**：典型"单写者 channel 推送 + 消费方 atomic 热替换"模式；Provider 本身无 goroutine、无锁，并发安全来自"建描述符时单线程写、之后只读"。基数树 `Lookup` 为只读并发安全。
- **控制器模式**：非 Reconcile；Provider 是事件源（channel 推送），消费方 codec 是事件处理器。文件 watch 未实现（TODO），热更新靠用户主动 `UpdateIDL`。
- **多二进制与部署边界**：纯运行时库，无独立二进制。
- **internal 边界与依赖方向**：descriptor 包依赖 dynamicgo（外部）与 `pkg/serviceinfo`（StreamingMode 常量）；Provider 依赖 descriptor；codec 依赖 Provider 接口（消费方定义 `DescriptorProvider` 在 `descriptor_provider.go:26`，实现在同包），依赖方向单向无环。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| IDL 描述符与 Provider 架构图 | `generic-descriptor-architecture.html` | architecture | showcase |
| IDL 解析到描述符推送数据流 | `generic-descriptor-dataflow.html` | dataflow | standard（dataflow 自动布局对横边标签间距严格，缩短标签并加宽 viewBox 后通过标准档；showcase 未过，按流程降档，render 退出码 0） |

- 本叶子不补 sequence/lifecycle 图：Provider 推送是单向管道（已用 dataflow 表达），无跨组件请求-响应时序，也无显式状态机（channel 开/关由 `sync.Once` 隐式管理）。
- JSON IR 源文件：`json/generic-descriptor-architecture.json`、`json/generic-descriptor-dataflow.json`。
