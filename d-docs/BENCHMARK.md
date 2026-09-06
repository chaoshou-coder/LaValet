# V Benchmark 策略

状态：**已锁定**

## 1. 首要目标

V 的首要目标是提高面试竞争力，以公开 benchmark 中可复现的速度优势展示工程能力。

```text
maximize InterviewSignal
        │
        ▼
公开 benchmark + 可复现结果 + 清晰归因
        │
        ▼
Terminal-Bench 2.1 + Harbor
```

生产通用性、安全、生态完整度均低于该目标。

## 2. 主战场

| 项目 | 决策 |
|---|---|
| Benchmark | `terminal-bench/terminal-bench-2-1` |
| Runner / telemetry | Harbor |
| 核心速度指标 | `agent_execution` E2E latency |
| 能力约束 | Terminal-Bench reward / accuracy |
| 统计 | p50 / p95 / p99 + success rate |
| 对照 | Pi 等公开 Agent，同任务、同环境 |

官方榜单仍以 accuracy 为主；V 额外利用 Harbor timing 建立 speed track。

## 3. 评价函数

单纯追求最短时间会奖励“快速失败”，因此速度必须带能力约束：

```text
minimize  T_agent_execution
subject to accuracy >= target
```

优先比较 Pareto frontier：

```text
accuracy
   ▲
   │        ●
   │     ●
   │  ●
   └────────────► latency
       lower is better
```

任何“更快”结论必须同时报告 accuracy。

## 4. 双赛道

### Rank Track｜最终系统刷榜

目标：在合法 benchmark 规则内，把完整 Agent 的 E2E 压到最低。

允许优化：

```text
reasoning effort / thinking
max output tokens
prompt / policy
工具描述与参数格式
上下文裁剪 / 压缩
减少无效文本输出
减少 model ↔ tool 轮数
并行 / batching / early dispatch
提前结束已完成任务
provider / transport / harness hot path
```

这里测的是 **V 作为完整系统**，不要求只隔离 harness。

### Harness Track｜归因实验

目标：证明 V 本身比其他 harness 更快。

必须固定：

```text
model
provider / endpoint
reasoning effort
max output tokens
sampling 参数
任务与环境
并发度
```

尽量固定 tool capability；无法完全等价时必须记录差异。

这里的核心 claim：

> Same model. Same config. Same tasks. Different harness.

## 5. Token / Thinking 策略

**减少 reasoning 与生成 token 是 P0 性能方向。**

原因：

```text
T_E2E ≈ Σ(model latency + tool latency + harness overhead)

model latency
  ∝ reasoning tokens
  + visible output tokens
  + model round trips
```

优先顺序：

```text
1. 删除无价值的自然语言输出
2. tool-first：能调用工具就不先解释
3. 限制每轮 visible output token
4. 测试 low / minimal / off reasoning
5. 用 benchmark 找 accuracy-latency Pareto 点
6. 只有收益成立才增加自适应 reasoning 机制
```

禁止凭直觉永久关闭 thinking。Terminal-Bench 包含困难长程任务，reasoning 降低可能增加失败、重试和 tool round-trip，最终反而提高 E2E。

## 6. 每次实验必须记录

| 维度 | 指标 |
|---|---|
| 结果 | reward / accuracy / error / timeout |
| E2E | `agent_execution` p50 / p95 / p99 |
| 模型 | prompt / cached / reasoning / output tokens（能取则取） |
| 轨迹 | model turns / tool calls / failed tool calls |
| Harness | TTFA / ITL / request prepare / dispatch |
| 配置 | model / effort / token limit / provider / commit SHA |

## 7. 开发优先级规则

新功能先问：

```text
能否改善 Terminal-Bench 2.1 的
accuracy ↔ latency Pareto frontier？
        │
   ┌────┴────┐
  YES       NO
   │         │
 高优先级   默认延后
```

生产价值不能单独构成进入核心的理由。

## 8. 官方依据

- Terminal-Bench 2.1 leaderboard：支持 Custom Agent，禁止修改任务 timeout/resources，并公开 Agent / Model / Effort / Accuracy / Tokens / Cost。
- Terminal-Bench 官方仓库：Harbor 运行时可传 agent、model 与 `reasoning_effort`。
- Harbor：trial/job 是正式评测单位，结果包含独立 phase timing，可区分 `agent_execution` 与 setup / verifier / total。
