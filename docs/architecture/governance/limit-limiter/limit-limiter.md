# 限流与 ACL（limit-limiter）

> 本文是 `governance` 域下的叶子子系统文档。域级总览见 `../governance.md`，本文只展开 `pkg/limit/`（限流配置抽象）、`pkg/limiter/`（并发/QPS 限流器实现）、`pkg/acl/`（访问控制规则），不展开限流中间件如何挂入 client/server 中间件链（见相邻 client/server 集成）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 限流配置 | `limit.Option`：MaxConnections/MaxQPS/UpdateControl | `pkg/limit/limit.go:25` |
| 动态更新接口 | `limit.Updater.UpdateLimit(*Option)` | `pkg/limit/limit.go:20` |
| 并发限流器接口 | `ConcurrencyLimiter.Acquire/Release/Status` | `pkg/limiter/limiter.go:28` |
| 速率限流器接口 | `RateLimiter.Acquire/Status` | `pkg/limiter/limiter.go:40` |
| 并发/连接实现 | `connectionLimiter`：原子计数信号量 | `pkg/limiter/connection_limiter.go:25,44` |
| QPS 令牌桶实现 | `qpsLimiter`：ticker 回补令牌 | `pkg/limiter/qps_limiter.go:28,72` |
| 限流器包装 | `NewLimiterWrapper`：组合并发+QPS 为 Updater | `pkg/limiter/limiter.go:61` |
| 可动态调参 | `Updatable.UpdateLimit(limit)` | `pkg/limiter/limiter.go:50` |
| 限流上报 | `LimitReporter.ConnOverloadReport/QPSOverloadReport` | `pkg/limiter/limiter.go:55` |
| 项级限流配置 | `LimiterConfig`（动态配置下发项） | `pkg/limiter/item_limiter.go:33` |
| ACL 中间件 | `NewACLMiddleware(rules []RejectFunc)` | `pkg/acl/acl.go:45` |
| ACL 规则执行 | `ApplyRules`：依次执行拒绝规则 | `pkg/acl/acl.go:32` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `limit.Option` | `limit.go:25` | 限流参数载体 |
| `limit.Updater` | `limit.go:20` | 限流配置动态更新抽象 |
| `ConcurrencyLimiter` | `limiter.go:28` | 并发/连接数限制接口 |
| `RateLimiter` | `limiter.go:40` | QPS 速率限制接口 |
| `connectionLimiter` | `connection_limiter.go:25` | 原子 `curr` 与 `lim`，Acquire 先+1 再判 |
| `qpsLimiter` | `qps_limiter.go:28` | 令牌桶：tokens 原子扣减，ticker 回补 |
| `limitWrapper` | `limiter.go:68` | 把两种限流器适配为 `limit.Updater` |
| `RejectFunc` | `acl.go:30` | 单条 ACL 拒绝规则 |

## 3. 关键调用链

1. **并发限流放行**：`connectionLimiter.Acquire`（`connection_limiter.go:44`）`x := atomic.AddInt32(&ml.curr, 1)`，若 `x > lim` 则退回并拒绝；成功占住后由 `Release`（`connection_limiter.go:51`）`AddInt32(&ml.curr, -1)` 释放。
2. **QPS 令牌扣减**：`qpsLimiter.Acquire`（`qps_limiter.go:72`）先读 `limit<=0` 直通，再读 `tokens<=0` 拒绝，否则 `atomic.AddInt32(&l.tokens, -1)` 扣减并判定。后台 `startTicker`（`qps_limiter.go:90`）按 interval 周期 `updateToken`（`qps_limiter.go:118`）回补令牌。
3. **动态调参**：配置源构造 `limit.Option` → `limitWrapper.UpdateLimit`（`limiter.go:73`）断言 `Updatable` 并 `UpdateLimit(MaxConnections/MaxQPS)`，原子 store 新阈值。
4. **ACL 判定**：`NewACLMiddleware` 返回中间件（`acl.go:45`），请求进来调 `ApplyRules`（`acl.go:32`）依次跑每条 `RejectFunc`，任一返回 reason 即拒绝。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `Option.MaxConnections` | 最大并发/连接数 | `limit.go:26` |
| `Option.MaxQPS` | 最大每秒请求数 | `limit.go:27` |
| `NewQPSLimiter(interval, limit)` | 令牌桶回补间隔与桶容量 | `qps_limiter.go:38` |
| `UpdateControl` | 配置源注入 Updater 的回调 | `limit.go:31` |
| `LimiterConfig` | 项级（按方法）限流配置 | `item_limiter.go:33` |

## 5. 错误与重试语义

- 限流拒绝不抛错，由中间件返回拒绝错误（如 `kerrors.ErrLimitExceeded` 类），不进入重试。
- 限流触发时调 `LimitReporter` 上报过载指标。
- qpsLimiter 的 `limit<=0` 表示不限流直通。
- 无自动重试；限流是"硬拒绝"语义。

## 6. 并发细节

- **原子操作**：connectionLimiter 与 qpsLimiter 全部用 `sync/atomic` 无锁，Acquire/Release 并发安全。
- **goroutine**：qpsLimiter 起一个后台 ticker goroutine（`startTicker`，`qps_limiter.go:90`），`stopTicker`（`qps_limiter.go:107`）用于关闭，防泄漏。
- **临界区**：无 mutex；阈值热更新用 atomic.Store。
- **context**：Acquire 接收 ctx 但当前实现不依赖 ctx 取消。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/limit/`、`pkg/limiter/`、`pkg/acl/`。

**Out-of-Scope（不在本仓库源码内）**
- 配置中心如何把 Option 推到 Updater（见 `../xds/xds.md` 与 configutil）。
- 限流中间件在 client/server 中间件链中的编织，归 client/server 集成（本分片不展开）。
- 过载上报的 metrics 后端见 `../stats-rpcinfo/stats-rpcinfo.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：client/server 把 ACL 中间件与限流检查挂入调用链入口。
- 本叶子 → 下游：拒绝时返回错误并上报；放行则进入正常 RPC 路径。
- 流向：请求 → ACL 规则 → 并发 Acquire → QPS Acquire → 下游。

## 9. 语言专项适配口径

- **并发模型**：纯 atomic 无锁限流器，是 Kitex 高性能路径的典型写法；唯一 goroutine 是 qps ticker，有明确 stop 边界。
- **控制器模式**：非 Reconcile；限流是同步判断式，状态由原子计数器表达。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：`pkg/limiter` 依赖 `pkg/limit` 与 `pkg/endpoint`；`pkg/acl` 依赖 `pkg/endpoint`。依赖方向单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 限流与 ACL 架构图 | `limit-limiter-architecture.html` | architecture | showcase |
| 限流判定时序 | `limit-limiter-sequence.html` | sequence | showcase（缩短参与者标签后通过） |

- 本叶子不补 lifecycle/dataflow：限流是无状态/计数器式判断，无阶段迁移；决策流已由时序图表达。
- JSON IR 源文件：`json/limit-limiter-architecture.json`、`json/limit-limiter-sequence.json`。
