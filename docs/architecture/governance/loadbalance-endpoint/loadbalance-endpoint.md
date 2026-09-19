# 负载均衡与 endpoint 选取（loadbalance-endpoint）

> 本文是 `governance` 域下的叶子子系统文档。域级总览见 `../governance.md`，本文只展开负载均衡算法族（`pkg/loadbalance`）与 endpoint 中间件/选取抽象（`pkg/endpoint`）在发现侧的接入；客户端中间件链如何编织进调用链归另一分片，本文不展开 `client/middlewares.go`。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 负载均衡接口 | `Loadbalancer`：`GetPicker(discovery.Result) Picker` + `Name()` | `pkg/loadbalance/loadbalancer.go:31` |
| 选实例接口 | `Picker`：`Next(ctx, request) discovery.Instance` | `loadbalancer.go:26` |
| 增量再平衡接口 | `Rebalancer`：`Rebalance(Change)` / `Delete(Change)` | `loadbalancer.go:37` |
| 默认加权均衡器 | `weightedBalancer` + 工厂 `NewWeightedBalancer/RoundRobin/Interleaved/Random/RandomWithAliasMethod` | `pkg/loadbalance/weighted_balancer.go:35,42` |
| 取 Picker 缓存与单飞 | `GetPicker` 按 `CacheKey` 查 `pickerCache`，未命中用 singleflight `sfg.Do` 只建一次 | `weighted_balancer.go:71,79` |
| 算法派发 | `createPicker` 过滤零权重、判断是否均衡（同权重），选择 RR/WRR/随机/别名/交错 WRR | `weighted_balancer.go:88,111` |
| 轮询 | `RoundRobinPicker`：原子计数器取模 | `weighted_round_robin.go:135,151` |
| 平滑加权轮询 | `WeightedRoundRobinPicker`：nginx 风格平滑 WRR，预计算 vnode，读快路径 RLock、建慢路径 Lock | `weighted_round_robin.go:68,80` |
| 加权随机 | `weightedRandomPicker` / `randomPicker` | `weighted_random.go:27,40` |
| 别名法加权随机 | `newAliasMethodPicker`（O(1) 按权重采样） | `weighted_random_with_alias_method.go`、`weighted_balancer.go:128` |
| 交错加权轮询 | `newInterleavedWeightedRoundRobinPicker` | `interleaved_weighted_round_robin.go` |
| 一致性哈希 | `consistPicker`：`KeyFunc` 提取请求键，`maphash` 哈希 + 虚拟节点环 + `sync.Map` 缓存结果 | `consist.go:43,129,153` |
| 原子轮询计数器 | `round`：`atomic.AddUint64`，64 字节缓存行对齐 | `iterator.go:26,32` |
| Picker 缓存 | `lbcache`：按目标缓存 Picker，支持 hookable 与 shared ticker | `pkg/loadbalance/lbcache/` |
| endpoint 抽象 | `Endpoint func(ctx, req, resp) error`、`Middleware`、`MiddlewareBuilder` | `pkg/endpoint/endpoint.go:22,25,28` |
| 中间件链编织 | `Chain`（反向包裹）、`Build`（递归包裹）、`DummyMiddleware/DummyEndpoint` | `endpoint.go:31,41,57,62` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Loadbalancer` | `loadbalancer.go:31` | 给定发现结果产出 Picker；按名字注册 |
| `Picker` | `loadbalancer.go:26` | 单次调用选一个 Instance；无状态算法用原子计数器，有状态算法读缓存 |
| `Rebalancer` | `loadbalancer.go:37` | 发现结果变化时增量更新 Picker 缓存而非整体重建 |
| `weightedBalancer` | `weighted_balancer.go:35` | 唯一具体 Loadbalancer，按 `kind` 决定算法族；持有 `pickerCache` 与 singleflight |
| `WeightedRoundRobinPicker` | `weighted_round_robin.go:68` | 平滑 WRR；vnode 数组懒构建，读写分离锁 |
| `consistPicker` / `consistBalancer` | `consist.go:129` | 一致性哈希选主节点；KeyFunc 从请求提取键 |
| `round` | `iterator.go:26` | 缓存行对齐的原子自增计数器，避免伪共享 |
| `Endpoint`/`Middleware`/`MiddlewareBuilder` | `endpoint.go:22,25,28` | RPC 调用链的函数式抽象；治理中间件（重试/熔断/限流）即实现 `Middleware` |
| `Chain`/`Build` | `endpoint.go:31,41` | 把中间件列表编织成单层 Middleware |

## 3. 关键调用链

1. **发现结果 → Picker**：客户端拿到 `discovery.Result` 调 `weightedBalancer.GetPicker(e)`（`weighted_balancer.go:71`）。若 `e.Cacheable` 为真，先 `pickerCache.Load(e.CacheKey)`（`weighted_balancer.go:77`），未命中用 `sfg.Do` 单飞创建并 `Store`（`weighted_balancer.go:79-83`）；`createPicker`（`weighted_balancer.go:88`）过滤零权重实例、判断是否同权重（`balance`），按 `kind` 派发具体 picker 构造器。
2. **一次选实例（平滑 WRR）**：`WeightedRoundRobinPicker.Next`（`weighted_round_robin.go:80`）用 `iterator.Next() % vcapacity` 取 vnode 下标；快路径 `RLock` 直接读预计算 vnode（`weighted_round_robin.go:84-86`）；若该槽尚未填充（慢路径），`Lock` 后批量 `buildVirtualWrrNodes`（`weighted_round_robin.go:92-102`），底层 `nextWrrNode`（`weighted_round_robin.go:116`）按 nginx 平滑 WRR 规则选当前权重最大节点。
3. **一致性哈希选主**：`consistPicker.Next`（`consist.go:153`）调 `cp.cb.opt.GetKey(ctx, request)` 提取键，`maphash.String(hashSeed, key)` 哈希后 `buildConsistResult` 在虚拟节点环上顺时针找主节点 `res.Primary`。
4. **发现增量驱动再平衡**：`weightedBalancer.Rebalance(change)`（`weighted_balancer.go:141`）在 `Cacheable` 时用新结果重建 Picker 并覆盖缓存；`Delete`（`weighted_balancer.go:149`）按 CacheKey 删缓存。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `client.WithLoadBalancer(lb)` | 注入 Loadbalancer；默认加权轮询 | `weighted_balancer.go` 工厂 |
| `NewWeightedRoundRobinBalancer` 等 | 选择 RR/交错 RR/随机/别名随机/一致性哈希 | `weighted_balancer.go:47,53,59,65` |
| `ConsistentHashOption.KeyFunc` | 从 ctx+request 提取哈希键 | `consist.go:46,79` |
| 零权重实例 | `createPicker` 跳过并 `klog.Warnf` 告警 | `weighted_balancer.go:95` |
| `Info.Weight` / `DefaultWeight=10` | 实例权重；同权重时退化到普通 RR/随机 | `registry.go:43`、`discovery.go:32` |

## 5. 错误与重试语义

- 实例列表为空时 `createPicker` 返回 `DummyPicker`（`weighted_balancer.go:108`），其 `Next` 返回 nil，由上层判定无可用实例。
- 一致性哈希 `KeyFunc` 返回空键时 `Next` 返回 nil（`consist.go:157`）。
- 本包不做 RPC 重试；重试属 retry 叶子（见 `../retry/retry.md`）。选到坏实例后的重试由 retry/熔断中间件在 endpoint 链外层处理。

## 6. 并发细节

- **原子计数器**：`round.Next()` 用 `atomic.AddUint64`（`iterator.go:33`），64 字节对齐避免伪共享；轮询类 picker 无锁并发选实例。
- **读写分离锁**：`WeightedRoundRobinPicker` 的 vnode 表用 `sync.RWMutex`（`weighted_round_robin.go:76`），读路径 RLock，懒构建慢路径 Lock；double-check 防止重复构建（`weighted_round_robin.go:94`）。
- **缓存单飞**：`GetPicker` 用 singleflight（`sfg.Do`，`weighted_balancer.go:79`）避免并发对同一 CacheKey 重复建 Picker。
- **一致性哈希结果缓存**：`consistInfo → sync.Map[hash] *consistResult`（`consist.go:37`），同键复用结果；`consistPicker` 经对象池 `Recycle` 复用（`consist.go:147`）。
- context：`Picker.Next(ctx, request)` 透传 ctx，一致性哈希用 ctx 提取键；LB 本身不传播超时。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/loadbalance/`：Loadbalancer/Picker/Rebalancer 接口、各算法实现、`lbcache`。
- `pkg/endpoint/`：Endpoint/Middleware/MiddlewareBuilder 与 Chain/Build；本叶子只讲它在发现侧（选实例）的接入与中间件抽象，不讲客户端中间件链编织。

**Out-of-Scope（不在本仓库源码内）**
- 实例数据源（注册/发现后端）见 `../registry-discovery/registry-discovery.md`。
- 客户端如何把 LB 选出来的 Instance 交给连接池建连、以及 `client/middlewares.go` 如何编织完整中间件链，属 client 分片。
- `pkg/endpoint/cep`、`pkg/endpoint/sep`（连接型/流式 endpoint 封装）与 `unary_endpoint.go` 属传输层适配，不在本叶子展开。

## 8. 与相邻子系统交互

- 上游 → 本叶子：发现侧（registry-discovery 叶子）产出 `discovery.Result/Change`；客户端调用链在每次请求时向 Picker 要 Instance。
- 本叶子 → 下游：`Picker.Next` 返回 `discovery.Instance` 供 `pkg/remote` 连接池选连接/建连；治理中间件（重试/熔断/限流，governance 其他叶子）以 `endpoint.Middleware` 形式包裹在选实例与实际调用之间。
- 流向：发现结果 → Loadbalancer.GetPicker → Picker.Next → Instance → 连接池。

## 9. 语言专项适配口径

- **并发模型**：典型"读多写少 + 原子计数器 + RWMutex 懒加载"。Picker 是热路径对象，选实例需无锁或读锁；vnode 预计算把运行时 O(n) 平滑 WRR 摊到建表与懒填充。singleflight 防止缓存击穿。
- **控制器模式**：非 Reconcile；`Rebalance/Delete` 是事件驱动增量更新（发现 Change 驱动），类似 informer 事件 → 缓存更新，但无 workqueue 重试。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：`pkg/endpoint` 是公开中间件抽象（治理各中间件与 client/server 都依赖它），`pkg/loadbalance` 依赖 `pkg/discovery`（Instance/Result）。依赖方向单向：endpoint 层被治理中间件与 RPC 核心依赖。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 负载均衡与 endpoint 选取架构图 | `loadbalance-endpoint-architecture.html` | architecture | showcase |
| 一次 RPC 选实例时序 | `loadbalance-endpoint-sequence.html` | sequence | showcase |

- 本叶子不补 dataflow/lifecycle 图：选实例是同步算法调用（已用 sequence 表达），无独立数据管道或状态机（Picker 内部状态仅计数器/缓存，由原子量与锁保护）。
- JSON IR 源文件：`json/loadbalance-endpoint-architecture.json`、`json/loadbalance-endpoint-sequence.json`。
