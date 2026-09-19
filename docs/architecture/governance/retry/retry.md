# 重试策略（retry）

> 本文是 `governance` 域下的叶子子系统文档。域级总览见 `../governance.md`，本文只展开 `pkg/retry/` 的失败重试、备份请求重试、混合重试策略与退避，不展开熔断器本身的状态机（见 `../circuitbreak-fallback/circuitbreak-fallback.md`，本叶子只与它交互）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 重试器接口 | `Retryer`：`IsRetryError/AllowRetry/Do/Prepare/UpdatePolicy/Dump` 等 | `pkg/retry/retryer.go:41` |
| 重试容器 | `Container`：按方法名缓存 Retryer，支持动态增删/更新策略 | `pkg/retry/retryer.go:190,236,261,277` |
| 失败重试 | `failureRetryer.Do`：串行循环调用，按 StopPolicy/ShouldResultRetry 决定是否继续 | `failure_retryer.go:78` |
| 备份请求重试 | `backupRetryer.Do`：主请求超时 `retryDelay` 后并发发起备份请求，取先到者 | `backup_retryer.go:90,104` |
| 混合重试 | `mixedRetryer`：失败重试 + 备份请求组合 | `mixed_retryer.go` |
| 策略构建 | `BuildFailurePolicy/BuildBackupRequest/BuildMixedPolicy` | `policy.go:51,59,67` |
| 停止策略 | `StopPolicy`：MaxRetryTimes/MaxDurationMS/DisableChainStop/DDLStop/CBPolicy | `policy.go:129` |
| 退避策略 | `BackOffPolicy`：none/fixed/random，cfg 键 fix_ms/min_ms/max_ms/initial_ms/multiplier | `policy.go:149,158` |
| 可重试结果判定 | `ShouldResultRetry`：ErrorRetryWithCtx/RespRetryWithCtx/NotRetryForTimeout | `policy.go:177` |
| 熔断联动 | `cbContainer`：重试期间与熔断器交互（GetKey/RecordStat/错误率阈值） | `retryer.go:205`、`failure_retryer.go:128` |
| 百分比限流 | `WithContainerEnablePercentageLimit`：限制重试占比 | `retryer.go:142`、`percentage_limit.go` |
| 方法级重试配置 | `RetryConfig`（item_retry.go）：动态配置下发的单方法重试项 | `item_retry.go:35` |
| 便捷构造器 | `NewFailurePolicy().WithMaxRetryTimes/WithFixedBackOff/WithRetryBreaker...` | `failure.go:39,61,84,103` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Policy` | `policy.go:76` | 顶层策略：Enable + Type(Failure/Backup/Mixed) + 三类子策略 |
| `FailurePolicy` | `policy.go:101` | 失败重试：StopPolicy + BackOffPolicy + RetrySameNode + ShouldResultRetry |
| `BackupPolicy` | `policy.go:115` | 备份请求：RetryDelayMS + StopPolicy + RetrySameNode |
| `Retryer` | `retryer.go:41` | 重试器抽象；三种实现 failure/backup/mixed |
| `Container` | `retryer.go:190` | 按方法缓存 Retryer 的容器，是 client 接入重试的入口 |
| `RPCCallFunc` | `retryer.go:34` | 单次 RPC 调用函数；重试器循环调用它 |
| `cbContainer` | `retryer.go:205` | 熔断套件持有：CBControl + CBPanel + 是否启用统计/百分比限流 |
| `BackOff`（initBackOff） | `failure_retryer.go:286` | 退避算法对象；None/Fixed/Random 三态 |

## 3. 关键调用链

1. **失败重试主循环**：`failureRetryer.Do`（`failure_retryer.go:78`）先 `RLock` 读 MaxDuration/MaxRetryTimes（`failure_retryer.go:79-85`），随后 `for i := 0; i <= retryTimes; i++`（`failure_retryer.go:99`）：非首次先查是否超 `maxDuration`（`failure_retryer.go:104`）、再 `ShouldRetry` 判定停止策略（`failure_retryer.go:108`），通过后调 `rpcCall(ctx, r, req, curResp)`（`failure_retryer.go:124`），结果用 `isRetryResult`（`failure_retryer.go:130`）判断是否需要继续；命中停止或成功即 break。循环结束 `shallowCopyResults` 拷贝响应并 `recordRetryInfo`。
2. **备份请求**：`backupRetryer.Do`（`backup_retryer.go:90`）建 `done chan resultWrapper`（`backup_retryer.go:102`），先发起主请求，`timer := time.NewTimer(retryDelay)`（`backup_retryer.go:104`）；定时到期后主请求仍未返回则并发发起备份请求，`select` 取第一个完成的结果，其余丢弃。
3. **策略热更新**：配置中心下发新策略 → `Container.NotifyPolicyChange(key, p)`（`retryer.go:277`）→ 对应 Retryer `UpdatePolicy(rp)`（`failure_retryer.go:142`），重新校验 StopPolicy 并 `initBackOff`（`failure_retryer.go:162`），运行中的请求仍用旧策略，新请求用新策略。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `FailurePolicy.StopPolicy.MaxRetryTimes` | 最大重试次数；0 不重试 | `policy.go:130` |
| `StopPolicy.MaxDurationMS` | 重试总时长上限，超时即停 | `policy.go:131` |
| `StopPolicy.CBPolicy.ErrorRate` | 重试触发熔断的错误率阈值，默认 0.1，最小样本 10 | `policy.go:144`、`policy.go:138` |
| `BackOffPolicy.BackOffType` | none/fixed/random；fixed 用 fix_ms，random 用 min_ms/max_ms | `policy.go:159` |
| `BackupPolicy.RetryDelayMS` | 主请求超时多久后发备份请求 | `policy.go:116` |
| `ShouldResultRetry.NotRetryForTimeout` | 特定场景禁用默认超时重试（非幂等请求） | `policy.go:189` |
| `RetrySameNode` | 重试是否走同一节点（默认换节点） | `policy.go:104,118` |

## 5. 错误与重试语义

- 重试器自身就是"重试"语义：`ShouldResultRetry` 由用户函数或默认规则（如超时可重试）判定；不可重试错误立即返回。
- panic 被 `recover` 转为错误（`failure_retryer.go:91-95`），不逃逸出重试循环。
- 熔断打开时 `circuitBreakerStop` 直接停止重试（`failure_retryer.go:249`），避免对故障节点雪上加霜。
- 每次失败 `circuitbreak.RecordStat`（`failure_retryer.go:128`）把错误计入熔断器统计。
- 本叶子不做"重试本身的退避失败重试"；退避只是 sleep。

## 6. 并发细节

- **goroutine 边界**：失败重试是单 goroutine 串行循环；备份重试在 `retryDelay` 到期时再起一个 goroutine 发备份请求（`backup_retryer.go`），通过 buffered `done` chan 收集结果（容量 = retryTimes+1，注释强调不能更小否则接收阻塞，`backup_retryer.go:101`）。
- **锁**：`failureRetryer` 的策略读写用 `RLock/RUnlock`（`failure_retryer.go:79,85`）配合 `UpdatePolicy` 写锁，支持运行时换策略。
- **定时器**：备份重试用 `time.NewTimer`，到期触发备份；需注意 Stop 防泄漏（由 Done chan 关闭收尾）。
- context：Do 接收 ctx，超时/取消由 ctx 传播——ctx 取消后循环应尽快退出；MaxDuration 是独立的重试总时长上限。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/retry/`：三种 Retryer、Policy 树、Container、退避、百分比限流、item 配置。

**Out-of-Scope（不在本仓库源码内）**
- 熔断器的具体滑动窗口/半开探测算法见 `../circuitbreak-fallback/circuitbreak-fallback.md`；本叶子只调用 `circuitbreak.RecordStat` 与 `cbContainer` 接口。
- 配置中心动态下发链路（xDS/配置源）如何把 Policy 推到 Container，见 `../xds/xds.md` 与 configutil。
- 连接池选节点（RetrySameNode 的反向——换节点）见 `../loadbalance-endpoint/loadbalance-endpoint.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：client 调用链在 endpoint 中间件位调用 `Container` 取 Retryer 并 `Do`。
- 本叶子 → 下游：`RPCCallFunc` 最终走到 LB 选实例 + 实际 RPC；与熔断器（circuitbreak-fallback）交互记录统计/读熔断状态。
- 流向：一次 RPC 失败 → ShouldRetry 判定 → 退避 sleep → 换节点再调 → 成功/耗尽。

## 9. 语言专项适配口径

- **并发模型**：失败重试串行（单 goroutine 循环），备份重试并行（主+备两路竞争 first-wins）。策略热切换用 RWMutex 读多写少。备份请求的 done chan 容量需 ≥ 调用次数，是关键并发不变量。
- **控制器模式**：非 Reconcile；这是调用级同步重试循环，状态机由"调用→判定→再调用/停止"构成，已用 lifecycle 图表达。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：`pkg/retry` 依赖 `pkg/circuitbreak`（熔断交互）与 `pkg/rpcinfo`；被 client endpoint 中间件层依赖。依赖方向单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 重试容器与策略架构图 | `retry-architecture.html` | architecture | showcase |
| 失败重试状态机 | `retry-lifecycle.html` | lifecycle | standard（lifecycle 布局对双向回边与标签间距严格，拉开列距 + labelDy 错位后通过标准档；showcase 未过，按流程降档，render 退出码 0） |

- 本叶子不补 sequence/dataflow：重试循环已由 lifecycle 表达状态迁移；备份请求的"主备竞争"在 MD 第 3 节文字描述，未单独出时序图。
- JSON IR 源文件：`json/retry-architecture.json`、`json/retry-lifecycle.json`。
