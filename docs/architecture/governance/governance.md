# 治理（governance）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 域职责

本域负责客户端/服务端的流量治理：服务发现与注册抽象、负载均衡与 endpoint 选取、重试策略、熔断与降级、限流与 ACL。这些能力以 endpoint 中间件形式挂入调用链，共同决定一次 RPC"路由到哪个实例、失败后怎么办、何时拒绝、何时降级"。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/状态机 | 职责一句话 |
|------|------|--------|-------------|-----------|
| registry-discovery | [registry-discovery.md](registry-discovery/registry-discovery.md) | [架构图](registry-discovery/registry-discovery-architecture.html) | [时序](registry-discovery/registry-discovery-sequence.html) | 注册中心与服务发现接口 |
| loadbalance-endpoint | [loadbalance-endpoint.md](loadbalance-endpoint/loadbalance-endpoint.md) | [架构图](loadbalance-endpoint/loadbalance-endpoint-architecture.html) | [时序](loadbalance-endpoint/loadbalance-endpoint-sequence.html) | 负载均衡算法与 endpoint 选取 |
| retry | [retry.md](retry/retry.md) | [架构图](retry/retry-architecture.html) | [状态机](retry/retry-lifecycle.html) | 失败/备份/混合重试与退避 |
| circuitbreak-fallback | [circuitbreak-fallback.md](circuitbreak-fallback/circuitbreak-fallback.md) | [架构图](circuitbreak-fallback/circuitbreak-fallback-architecture.html) | [状态机](circuitbreak-fallback/circuitbreak-fallback-lifecycle.html) | 熔断器与降级 fallback |
| limit-limiter | [limit-limiter.md](limit-limiter/limit-limiter.md) | [架构图](limit-limiter/limit-limiter-architecture.html) | [时序](limit-limiter/limit-limiter-sequence.html) | 并发/QPS 限流与 ACL |

## 3. 域级机制细节

- 各治理能力均以 `endpoint.Middleware` 形式编织，调用顺序构成治理链。
- 熔断与重试协同：熔断打开时重试 `circuitBreakerStop` 立即停止；限流/熔断用 atomic 无锁或外部 Panel 状态机。
- 策略多支持动态热更新（atomic/RWMutex 读多写少）。

## 4. 域级图（可选）

本域不单独出域级架构图，由各叶子图覆盖。
