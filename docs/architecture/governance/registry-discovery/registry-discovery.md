# 注册中心与服务发现抽象（registry-discovery）

> 本文是 `governance` 域下的叶子子系统文档。域级总览见 `../governance.md`，本文只展开服务注册（`pkg/registry`）与服务发现（`pkg/discovery`）的接口定义与默认实现，不展开负载均衡如何消费实例列表（见 `../loadbalance-endpoint/loadbalance-endpoint.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 服务注册抽象 | `Registry` 接口：`Register(*Info)` / `Deregister(*Info)`，服务端经 `server.WithRegistry` 注入 | `pkg/registry/registry.go:28` |
| 注册信息结构 | `Info`：ServiceName/Addr/PayloadCodec/Weight/StartTime/WarmUp/Tags/SkipListenAddr | `pkg/registry/registry.go:35` |
| 空注册实现 | `NoopRegistry`：Register/Deregister 恒返回 nil，作为默认值 | `pkg/registry/registry.go:55,60` |
| 服务发现抽象 | `Resolver` 接口：`Target/Resolve/Diff/Name`，客户端经 `client.WithResolver` 注入 | `pkg/discovery/discovery.go:56` |
| 发现结果 | `Result`：Cacheable/CacheKey/Instances；`Change`：Added/Updated/Removed 增量 | `discovery.go:37,48` |
| 实例抽象 | `Instance` 接口：Address/Weight/Tag；`NewInstance` 构造默认实例 | `discovery.go:177,131` |
| 默认 Diff 实现 | `DefaultDiff`：按地址字符串集合差集计算 Added/Updated/Removed | `discovery.go:73` |
| 函数式 Resolver 合成 | `SynthesizedResolver`：用 TargetFunc/ResolveFunc/DiffFunc/NameFunc 四件套拼装一个 Resolver | `discovery.go:140` |
| 默认权重 | `DefaultWeight = 10` | `discovery.go:32` |
| 发现事件名常量 | `ChangeEventName="discovery_change"`、`DeleteEventName="discovery_delete"` | `discovery/constants.go:21` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Registry` | `registry.go:28` | 注册扩展点；后端实现（etcd/zk/consul）实现此接口后注入 server |
| `Info` | `registry.go:35` | 注册负载；Weight/WarmUp 供 LB 与预热使用，Tags 承载扩展元数据 |
| `Resolver` | `discovery.go:56` | 发现扩展点；`Target` 把 EndpointInfo 映射为缓存键，`Resolve` 取实例列表，`Diff` 算增量 |
| `Instance` | `discovery.go:177` | 单个服务实例的最小视图（地址+权重+标签） |
| `Result`/`Change` | `discovery.go:37,48` | 发现结果与增量；`Change` 用于驱动 LB 增量更新与 stats 事件 |
| `SynthesizedResolver` | `discovery.go:140` | 让用户用函数而非整接口实现 Resolver；`DiffFunc` 为 nil 时回落 `DefaultDiff` |

## 3. 关键调用链

1. **服务端注册**：server 启动时拿到用户注入的 `Registry`，对每个服务构造 `Info{ServiceName, Addr, PayloadCodec, Weight, WarmUp, Tags}`（`registry.go:35`），调 `Registry.Register(info)`（`registry.go:29`）把实例写入注册中心；优雅退出时 `Deregister(info)`（`registry.go:30`）摘除。未注入时用 `NoopRegistry`（`registry.go:55`），注册调用空转。
2. **客户端发现（冷启动）**：客户端调 `Resolver.Target(ctx, endpointInfo)`（`discovery.go:58`）把目标服务描述成可缓存的 description；再 `Resolve(ctx, desc)`（`discovery.go:61`）拿 `Result{Instances}`。
3. **增量更新**：新一次 `Resolve` 后，客户端调 `Resolver.Diff(cacheKey, prev, next)`（`discovery.go:66`）；默认实现 `DefaultDiff`（`discovery.go:73`）用地址字符串做 map 差集——prev 有 next 无则 `Removed`，next 有 prev 无则 `Added`，同地址但权重变了则 `Updated`（`discovery.go:92-96`），返回 `(Change, 是否变化)`。`Change` 随后驱动 LB 更新内部实例集并发 `discovery_change` 事件。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `server.WithRegistry(reg)` | 注入注册实现；不注入则 `NoopRegistry` | `registry.go` 注释 |
| `client.WithResolver(r)` | 注入发现实现；不注入则用直连地址 | `discovery.go` 注释 |
| `Info.Weight` / `DefaultWeight` | 注册实例权重默认 10，供加权 LB 使用 | `registry.go:43`、`discovery.go:32` |
| `Info.WarmUp` | 预热时长，配合 warmup 控制流量逐步放大 | `registry.go:45` |
| `Info.Tags` / `Instance.Tag` | 扩展标签（环境/机房/集群），供路由与过滤 | `registry.go:48`、`discovery.go:121` |

## 5. 错误与重试语义

- `Register/Resolve` 返回 error 直接透传；本包不重试。注册失败由 server 启动流程决定是否中断；发现失败由客户端 LB 走降级/已有缓存。
- `Resolve` 返回 error 时客户端通常沿用上一次缓存实例（`Result.Cacheable=true` 时 LB 可缓存）。
- `DefaultDiff` 不判错；标签变化当前 FIXME 未纳入 Updated 判定（`discovery.go:91`），仅按地址+权重判定。
- 重试语义属 governance 域 retry 叶子（见 `../retry/retry.md`）。

## 6. 并发细节

- 本包是纯接口 + 无状态默认实现（`DefaultDiff`、`NewInstance`、`SynthesizedResolver`），自身不起 goroutine、无锁。
- `Instance` 是不可视数据（构造后只读），可被多 goroutine 共享；`Tags` map 由构造方负责不再修改。
- 实际发现后端（watch 长连接、定时拉取、事件推送）在各后端实现里，不在本仓库；`Change` 事件经 stats 事件总线广播（见 `../stats-rpcinfo/stats-rpcinfo.md`）。
- context：`Target/Resolve` 均接收 `context.Context`，后端可据此做超时/取消传播。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/registry/registry.go`：注册接口与 Info/Noop。
- `pkg/discovery/discovery.go`、`constants.go`：发现接口、实例、Result/Change、DefaultDiff、SynthesizedResolver。

**Out-of-Scope（不在本仓库源码内）**
- 注册中心/发现后端实现（etcd、 ZooKeeper、Consul、境内注册中心等）——实现 `Registry`/`Resolver` 接口，不在本仓库。
- 负载均衡算法与 endpoint 选取见 `../loadbalance-endpoint/loadbalance-endpoint.md`。
- 客户端如何把 Resolver 结果接入调用链、连接池如何按实例建连，属 client/ 与 pkg/remote 分片。

## 8. 与相邻子系统交互

- 上游（服务端注册侧）：`server/` 用 `Registry.Register/Deregister` 上报实例。
- 上游（客户端发现侧）：`client/` 与 endpoint 中间件调用 `Resolver.Target/Resolve/Diff`。
- 下游：`Resolver.Resolve` 产出的 `[]Instance` 喂给负载均衡器（loadbalance-endpoint 叶子）做选实例；`Change` 经事件总线通知 stats。
- 流向：server→注册后端；client→发现后端→实例列表→LB→连接池。

## 9. 语言专项适配口径

- **并发模型**：本包无并发原语；并发语义由后端实现与消费方（LB、事件总线）承担。`Instance` 不可变，天然并发安全。
- **控制器模式**：非 Reconcile；这是"接口定义层"，真正的 watch→缓存→diff→更新管道在 LB 与后端实现中。
- **多二进制与部署边界**：纯运行时接口库，无独立二进制。
- **internal 边界与依赖方向**：接口定义在消费侧（`pkg/registry`、`pkg/discovery` 是公开扩展点），后端实现作为独立模块依赖本包——典型依赖倒置。`discovery` 依赖 `pkg/rpcinfo`（EndpointInfo）与 `pkg/utils`（NewNetAddr），方向单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 注册中心与服务发现抽象架构图 | `registry-discovery-architecture.html` | architecture | showcase |
| 服务发现解析时序 | `registry-discovery-sequence.html` | sequence | showcase |

- 本叶子不补 dataflow/lifecycle 图：注册/发现是接口契约而非数据管道；实例集合的"上线/下线"状态由后端与 LB 维护，本包只产出增量 `Change`，不单独建模状态机。
- JSON IR 源文件：`json/registry-discovery-architecture.json`、`json/registry-discovery-sequence.json`。
