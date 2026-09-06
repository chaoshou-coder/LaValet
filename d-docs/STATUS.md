# La Valet（V）开发状态与验收门槛

状态：**已锁定流程**

当前状态：**S1｜最小可运行 Agent**

本文件定义项目开发阶段、进入下一阶段的验收门槛，以及状态切换规则。当前具体工作只记录在 [`TODOS.md`](./TODOS.md)。Benchmark 规则仍以 [`BENCHMARK.md`](./BENCHMARK.md) 为唯一权威位置。

## 状态流

```text
S0 立项锁定      已完成
 │
 ▼
S1 最小可运行 Agent      ← 当前
 │
 ▼
S2 Benchmark 闭环
 │
 ▼
S3 Baseline 冻结
 │
 ▼
S4 E2E 速度突破
 │
 ▼
S5 Harness 归因
 │
 ▼
S6 性能收敛
 │
 ▼
S7 可展示
```

## 状态切换规则

1. 只有当前状态的验收门槛全部满足，才能进入下一状态。
2. 未通过当前 Gate，不提前建设后续阶段机制。
3. 状态切换时更新本文件的“当前状态”，并将 `TODOS.md` 完整替换为新状态的工作。
4. Benchmark 结果、实现存在与否必须由代码或实际运行结果证明，不能由计划文档视为已完成。

## S0｜立项锁定

定义：项目目标和核心边界已经确定，可以停止继续选题。

验收门槛：

- 首要目标：提高面试竞争力
- 主战场：Terminal-Bench 2.1
- Runner / telemetry：Harbor
- 首要 baseline：Pi
- Runtime：Bun + TypeScript
- 平台：Linux + macOS 公共交集
- Inference：外部 Provider
- 核心工具：`read / write / edit / bash`
- 非目标与极简原则已经锁定

状态：**已通过**。

## S1｜最小可运行 Agent

定义：得到一个足够小、功能正确的 V；暂不要求 benchmark 能力或速度优势。

验收门槛：

- Bun + TypeScript 项目可运行
- 最小 Provider 可以完成一次真实模型调用并接收 streaming 输出
- `read / write / edit / bash` 四个工具可独立正确执行
- 模型 tool call 可以被解析、执行并回注 observation
- Agent loop 可以连续执行多轮工具调用并正常结束
- session 仅使用进程内存
- 核心执行路径保持单进程
- Linux 与 macOS 使用同一套核心实现并完成 smoke test

通过后进入 **S2**。

## S2｜Benchmark 闭环

定义：V 已能被 Harbor 稳定执行和测量，但还没有可信的相对性能结论。

验收门槛：

- Harbor Custom Agent adapter 可运行 V
- Terminal-Bench 2.1 单任务跑通
- 固定开发任务集可重复运行
- 多任务 job 可运行
- `result.json`、trajectory、token/timing 数据可保存
- `agent_execution`、reward / accuracy 可自动提取
- benchmark 环境、并发度、provider/model 配置可冻结
- Pi 能在同一评测条件下运行

通过后进入 **S3**。

## S3｜Baseline 冻结

定义：建立不可变的性能零点，后续所有优化都能相对此版本归因。

验收门槛：

- 保存 V commit SHA、Pi version、任务集和完整运行配置
- 保存 V / Pi 原始 benchmark 结果
- 得到 reward / accuracy、`agent_execution`、model turns、tool calls、token usage 基线
- 固定 Harness Track 的同模型同配置比较条件
- 根据 baseline 锁定 Rank Track 的 accuracy floor
- 根据 baseline 锁定第一阶段需要达到的 E2E speedup 目标

通过后进入 **S4**。

## S4｜E2E 速度突破

定义：围绕 Terminal-Bench 2.1 的真实任务时间进行主动优化，取得具有展示价值的 Rank Track 优势。

优化顺序遵循 [`BENCHMARK.md`](./BENCHMARK.md)：

```text
model round trips
→ reasoning / output tokens
→ context / observation tokens
→ parallel / batching / early dispatch
→ provider / shell / harness hot path
```

验收门槛：

- reward / accuracy 不低于 S3 锁定的 floor
- 达到 S3 锁定的 E2E speedup 目标
- 提升在重复运行中稳定存在
- p50 明确改善，p95 / p99 无不可接受退化
- 结果可由固定配置和 commit 复现

通过后进入 **S5**。

## S5｜Harness 归因

定义：解释 V 为什么快，区分模型配置、轨迹效率和 harness hot path 的贡献。

验收门槛：

- 完成 Harness Track：same model / endpoint / reasoning / token budget / sampling / tasks / concurrency
- 对比 V / Pi 的 `agent_execution`、TTFA、ITL、turns、tool calls、tokens
- 能解释主要速度差来自哪些机制
- 关键优化至少有独立 A/B 或 ablation 证据
- 记录无法完全控制的比较差异和无效实验

通过后进入 **S6**。

## S6｜性能收敛

定义：主要瓶颈已经被量化，继续增加复杂度的边际收益开始下降。

只有 profiler 证明必要时，才允许 persistent shell、session transport、delta transport、UDS、binary/custom protocol 等 P2 机制进入核心。

验收门槛：

- 主要 E2E 时间构成和瓶颈已知
- 尚存候选优化均有测量依据
- 高价值低复杂度优化已经完成
- 高复杂度候选已有“保留 / 拒绝”结论和数据依据
- 没有仅凭理论收益进入核心的机制

通过后进入 **S7**。

## S7｜可展示

定义：项目已经形成完整、可复现、可解释的工程证据链。

验收门槛：

```text
问题
→ baseline
→ profiling
→ 假设
→ 实现
→ A/B / ablation
→ E2E speedup
→ attribution
```

必须具备：

- 可复现的 Terminal-Bench 2.1 测试方法
- V vs Pi Rank Track 结果
- Harness Track 归因结果
- 核心架构与关键优化说明
- 原始结果、配置和版本可追溯
- README / benchmark report 能独立说明“为什么快、快多少、代价是什么”

达到 S7 后，第一阶段项目完成。
