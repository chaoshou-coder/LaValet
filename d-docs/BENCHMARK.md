# V Benchmark 策略

状态：**已锁定**

## 1. 首要目标

V 的首要目标是提高面试竞争力，并通过真实 Agent 任务中的显著速度优势展示工程能力。

```text
maximize 可展示的速度优势
          │
          ▼
Terminal-Bench 2.1 + Harbor
          │
          ├─ reward / accuracy 保证任务真的完成
          └─ agent_execution 衡量真实 E2E latency
```

选择 Terminal-Bench 2.1 的主要原因是：任务真实、多轮 Agent 交互充分、Harbor 可直接记录完整执行时间、无需额外建设评测体系。

生产通用性、安全、生态完整度同样低于速度目标。

## 2. Benchmark 选择冻结

**主战场固定为 Terminal-Bench 2.1 + Harbor。**

这不是因为它必须是最权威或理论上最适合 V 的 benchmark，而是因为继续比较、筛选、适配其他 benchmark 会产生与项目首要目标无关的准备成本。

```text
更多 benchmark 调研
       ↓
更多 adapter / 环境 / 规则工作
       ↓
更晚获得 baseline
       ↓
更晚开始速度迭代
```

因此：

```text
Terminal-Bench 2.1 = 固定开发靶场
Harbor             = 固定测量工具
Pi                 = 首要对照
```

默认不做：

```text
benchmark landscape 调研
为寻找更高权威性而切换 benchmark
同时维护多个旗舰 benchmark
为 benchmark 选择设计复杂评分体系
```

只有 Terminal-Bench 2.1 无法继续承载真实 E2E 速度比较时，才重新打开 benchmark 选择问题。

## 3. 主战场

| 项目 | 决策 |
|---|---|
| Benchmark | `terminal-bench/terminal-bench-2-1` |
| Runner / telemetry | Harbor |
| 核心目标 | 最大化真实任务 E2E 速度优势 |
| 核心指标 | `agent_execution` latency |
| 能力约束 | reward / accuracy |
| 统计 | p50 / p95 / p99 + success rate |
| 首要对照 | Pi |

```text
真实任务
  +
同任务 / 同环境
  +
尽可能公平的模型配置
  ↓
比较 V 与 Pi 的 E2E
```

官方 leaderboard 的排序方式不是项目目标；Harbor timing 才是主要数据源。

## 4. 评价函数

单纯追求最短时间会奖励快速失败，因此：

```text
minimize  T_agent_execution
subject to accuracy >= target
```

优化目标是把可接受 accuracy 下的 latency 压到最低。

```text
accuracy
   ▲
   │        ●
   │     ●
   │  ●  ← 优先寻找左上方
   └────────────► latency
       lower is better
```

任何速度结果必须同时报告 accuracy / reward。

## 5. 双赛道

### Rank Track｜最终速度

目标：让完整 V 系统在 Terminal-Bench 2.1 上尽可能快。

允许使用所有符合任务规则的优化：

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

这里优化的是 **完整 Agent E2E**，不是纯 harness overhead。

### Harness Track｜速度归因

目标：证明速度优势不只来自关闭 thinking 或缩短输出。

固定：

```text
model
provider / endpoint
reasoning effort
max output tokens
sampling 参数
任务与环境
并发度
```

尽量统一 tool capability。

核心 claim：

> Same model. Same config. Same tasks. Different harness.

Harness Track 服务于解释 Rank Track，不与 Rank Track 争夺开发优先级。

## 6. E2E 优化优先级

真实任务下，优先优化对 E2E 影响最大的量：

```text
P0  减少 model round trips
P0  减少 reasoning tokens
P0  减少 visible output tokens
P0  减少 context / observation tokens
P0  tool-first / final-only prose

P1  并行工具 / batching / early dispatch
P1  provider/session/connection reuse
P1  shell / FS hot path

P2  毫秒级 harness 微优化
```

原因：

```text
T_E2E ≈ Σ(model latency + tool latency + harness overhead)

少一次 model call
通常远大于
少几毫秒 Bun / dispatch 开销
```

## 7. Token / Thinking 策略

减少 reasoning 与生成 token 是 P0。

```text
1. 删除工具调用前的解释性 prose
2. 中间轮优先只输出 tool call
3. 限制 visible output token
4. sweep low / minimal / off reasoning
5. 找 accuracy ↔ latency Pareto 点
6. 只有静态配置不够时才考虑 adaptive reasoning
```

不预设 thinking 越少越好。reasoning 过低可能导致失败、重试和更多 tool round-trip，最终增加 E2E。

## 8. 每次实验记录

| 维度 | 指标 |
|---|---|
| 结果 | reward / accuracy / error / timeout |
| E2E | `agent_execution` p50 / p95 / p99 |
| 模型 | prompt / cached / reasoning / output tokens |
| 轨迹 | model turns / tool calls / failed tool calls |
| Harness | TTFA / ITL / request prepare / dispatch |
| 配置 | model / effort / token limit / provider / commit SHA |

## 9. 开发决策规则

新功能先问：

```text
能否明显降低 Terminal-Bench 2.1 的 E2E
且不让 accuracy 跌出可接受范围？
          │
     ┌────┴────┐
    YES       NO
     │         │
   优先做     延后
```

不再因为以下原因增加准备工作：

```text
“这个 benchmark 更权威”
“生产以后可能需要”
“其他 Agent 都有”
“理论上更完整”
```

原则：**固定 Terminal-Bench 2.1，尽快跑 baseline，然后围绕真实 E2E 数据迭代。**
