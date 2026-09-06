# V Benchmark 策略

状态：**已锁定**

本文件只记录最终决策及其原因，不记录中间讨论过程。

## 1. 目标

| 最终决策 | 原因 |
|---|---|
| V 首要追求真实 Agent 任务中的显著 E2E 速度优势 | 项目首要用途是提高面试竞争力；量化、可复现的速度差比生产完整度更容易直接展示工程能力 |
| 速度优化必须受 reward / accuracy 约束 | 单独最小化 latency 会奖励快速失败，无法证明 Agent 真正完成任务 |
| benchmark 权威性是低优先级 | 当前目标是展示明显速度优势，不需要为更高权威性增加额外准备成本 |

评价函数：

```text
minimize  T_agent_execution
subject to accuracy >= acceptable_target
```

## 2. 固定主战场

| 最终决策 | 原因 |
|---|---|
| Benchmark 固定为 `terminal-bench/terminal-bench-2-1` | 任务是真实 terminal Agent 工作，包含多轮 model ↔ tool 交互，能够体现完整 Agent E2E |
| Runner / telemetry 固定为 Harbor | 可直接运行 Custom Agent，并记录任务执行结果与 `agent_execution` timing，无需自建完整评测框架 |
| Pi 作为首要对照 Agent | Pi 属于成熟 coding harness 中的极简代表，与 V 的四工具方向接近，适合作为直接 baseline |
| 默认不再调研、筛选其他 benchmark | benchmark landscape、adapter、环境和规则比较会推迟 baseline 与实际速度迭代 |
| 只有 Terminal-Bench 2.1 无法继续承载真实 E2E 比较时才重新选 benchmark | 防止 benchmark 选择本身演化成独立项目 |

固定关系：

```text
Terminal-Bench 2.1 = 开发靶场
Harbor             = 测量工具
Pi                 = 首要对照
```

## 3. 核心指标

| 指标 | 用途 | 原因 |
|---|---|---|
| `agent_execution` | 主速度指标 | 最接近完整 Agent 实际执行阶段的 E2E latency |
| reward / accuracy | 能力约束 | 防止通过少做、失败或提前退出获得虚假速度优势 |
| p50 / p95 / p99 | 延迟分布 | 避免平均值掩盖长尾 |
| model turns | 速度归因 | 少一次 model call 往往比毫秒级 harness 优化收益更大 |
| reasoning / output / prompt tokens | 速度归因 | 模型生成和上下文处理通常占真实任务主要时间 |
| tool calls / failed tool calls | 轨迹效率 | 反映无效往返、重试和工具策略成本 |
| TTFA / ITL | harness 归因 | 用于解释首动作和多轮工具链中的固定开销 |

## 4. 双赛道

### Rank Track

**最终决策：Rank Track 是主目标，允许优化完整 Agent 系统中的所有合法速度变量。**

允许包括：

```text
reasoning effort / thinking
output token budget
prompt / policy
tool schema
context / observation 压缩
减少 model round trips
减少无效 prose
并行 / batching / early dispatch
provider / transport / harness hot path
```

原因：最终展示的是完整 Agent 完成真实任务需要多久；如果只优化 harness 内部毫秒级 overhead，会忽略更大的 E2E 成本来源。

### Harness Track

**最终决策：保留 Harness Track，只用于速度归因。**

固定：

```text
same model
same endpoint
same reasoning effort
same token budget
same sampling
same tasks / environment
same concurrency
```

核心比较：

> Same model. Same config. Same tasks. Different harness.

原因：需要证明 Rank Track 的速度优势不只是来自关闭 thinking、缩短输出或改变模型配置。

## 5. E2E 优化优先级

最终优先级：

```text
P0  减少 model round trips
P0  减少 reasoning tokens
P0  减少 visible output tokens
P0  减少 context / observation tokens
P0  tool-first / final-only prose

P1  并行工具 / batching / early dispatch
P1  provider / session / connection reuse
P1  shell / FS hot path

P2  毫秒级 harness 微优化
```

原因：

```text
T_E2E ≈ Σ(model latency + tool latency + harness overhead)
```

真实 Agent 任务中，减少一次模型调用或大量 token 通常比削减几毫秒 Bun / dispatch 开销产生更大的 E2E 收益。

## 6. Thinking / Token 策略

| 最终决策 | 原因 |
|---|---|
| reasoning effort 和 token budget 是 P0 优化变量 | reasoning / decode 直接占用 wall-clock |
| 中间轮采用 tool-first，尽量不输出解释性 prose | 无价值自然语言会增加 decode latency，却不推进任务 |
| 最终答案前尽量保持 final-only prose | 将 token 预算集中到真正需要面向用户输出的阶段 |
| 对 reasoning 做 `off/minimal/low/...` sweep，而不预设越低越好 | reasoning 过低可能增加错误、重试和 model round trips，反而提高 E2E |
| 暂不优先实现复杂 adaptive reasoning | 在静态配置收益尚未测清前，引入 router/classifier 会增加不必要复杂度 |

## 7. 开发规则

任何新机制先回答：

```text
能否明显降低 Terminal-Bench 2.1 的 E2E
且不让 accuracy 跌出可接受范围？
          │
     ┌────┴────┐
    YES       NO
     │         │
   优先做     延后
```

不因为以下理由增加工作：

```text
benchmark 更权威
生产以后可能需要
其他 Agent 都有
理论上更完整
```

最终开发循环：

```text
最小 V
  ↓
Harbor adapter
  ↓
V baseline + Pi baseline
  ↓
Harbor timing / trajectory
  ↓
优化最大 E2E 瓶颈
  ↓
重跑
  ↓
循环
```
