# Le Valet（V）项目决策

> 旧代号：`delta-agent` ｜ 中文名：拉挽乐 ｜ 仓库：`chaoshou-coder/LaValet`

## 1. 首要目标

**V 的首要目标是提高面试竞争力：在公开 Agent benchmark 中做出可复现、可解释的速度优势。**

```text
面试竞争力
   │
   ▼
公开 benchmark 成绩
   │
   ▼
Terminal-Bench 2.1 + Harbor
   │
   ├─ accuracy / reward 约束能力
   └─ agent_execution 衡量 E2E latency
```

开发优先级：

```text
benchmark 速度 / 可复现结果
            ↓
极简 / 可解释 / 可归因
            ↓
生产通用性 / 安全 / 生态完整度
```

详细评测协议见 [`BENCHMARK.md`](./BENCHMARK.md)。

## 2. 一句话定义

**V 是一个面向 Terminal-Bench 速度优化、运行于 Linux + macOS 公共交集上的极简 Agent Harness。**

```text
用户
 │
 ▼
V / Bun
 ├─ Agent loop
 ├─ Provider
 └─ Tools: read / write / edit / bash
        │
        ├──────────────► 本地 LLM API
        └──────────────► 云端 LLM API
```

模型推理在 V 之外完成；local LLM 只是 API provider 的一种。

## 3. 已锁定边界

| 项目 | 决策 | 含义 |
|---|---|---|
| Benchmark | Terminal-Bench 2.1 + Harbor | 开发与性能验证主战场 |
| 首要指标 | E2E latency | 优先优化 `agent_execution` |
| 能力约束 | reward / accuracy | 防止“快速失败”污染速度结论 |
| 平台 | Linux + macOS 公共交集 | 不做 macOS 专属核心 |
| Runtime | Bun | 优先直接使用 Bun 原语 |
| Inference | 外部 API | 不内置推理 runtime |
| Provider | 本地 / 云端统一看待 | local LLM 无特殊架构地位 |
| 核心工具 | `read/write/edit/bash` | 默认不增加第 5 个工具 |
| 安全 | 非核心目标 | 可为简单性与 benchmark 收益让位 |
| 核心原则 | 极简 + benchmark-first | 新机制必须证明 benchmark 价值 |

## 4. 最小执行模型

```text
input
  │
  ▼
agent loop ──► model API
                 │
                 ▼
              tool call
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
 read/write     edit       bash
      └──────────┼──────────┘
                 ▼
            observation
                 │
                 └──────────► model API ──► ... ──► final
```

当前 baseline：

```text
1 个 Bun 进程
+ 内存 session
+ 直接 spawn
+ 外部 model API
+ 4 个工具
```

默认没有：

```text
sandbox / daemon / worker pool / database / scheduler
plugin runtime / 内部 IPC / capability system / 内置 inference
```

## 5. 极简规则

```text
函数           > class hierarchy
单进程         > IPC
Bun / OS 原语  > framework
现有 Unix 工具 > harness 内重写
内存状态       > database
现有协议       > 自定义协议
Terminal-Bench 实测 > 假想生产需求
```

约束：

1. Bun 已解决的平台差异，不再包一层 platform abstraction。
2. 新组件优先由 Terminal-Bench / Harbor 数据证明必要性。
3. 平台专属 fast path 只有 benchmark 收益成立后才进入候选。
4. 不因为成熟 Agent 框架“都有”某功能就引入它。
5. 能改善 accuracy ↔ latency Pareto frontier 的优化优先。

## 6. Inference 边界

```text
                   ┌─ OpenAI / Anthropic / ...
V Provider Layer ──┤
                   ├─ llama.cpp server
                   ├─ BaseRT server
                   └─ 其它兼容 API
```

核心不依赖：

`CUDA / Metal / ANE / MLX / llama.cpp / BaseRT / vLLM / 特定模型`

当前不负责：模型 kernel、GPU/ANE 调度、显存/统一内存管理、推理 runtime 实现。

模型侧参数仍可作为 **Rank Track** 的 benchmark 优化变量，包括 reasoning effort、token budget 等；**Harness Track** 必须锁定这些参数做公平归因。

## 7. 平台边界

V 使用 **Linux 与 macOS 的实际公共能力子集**，不假设二者互为超集。

优先级：

```text
Bun 跨平台 API
      ↓
Unix/POSIX 公共原语
      ↓
平台专属能力（仅在 benchmark 收益被证明后）
```

目标是尽量让平台差异不进入核心。

## 8. 性能目标

核心指标是 Terminal-Bench 任务完整执行延迟：

| 指标 | 含义 |
|---|---|
| `agent_execution` | Harbor 中 Agent 实际执行阶段 E2E |
| TTFA | 输入 → 第一个有效动作 |
| ITL | 相邻工具动作之间的延迟 |
| p50/p95/p99 | 延迟分布 |
| reward / accuracy | 速度优化的能力约束 |
| tokens / turns | 解释 E2E 差异的重要变量 |

```text
T_E2E = model + harness + tool + observation + ... + final
```

优化对象包括 harness overhead，也包括减少 reasoning/output token、减少无效 model turn、减少 tool round-trip。

## 9. Benchmark 双赛道

```text
Rank Track
└─ 追求完整系统最低 E2E
   ├─ 可调 reasoning effort
   ├─ 可调 token budget
   ├─ 可优化 prompt / policy
   └─ 可优化 harness / provider / tools

Harness Track
└─ 证明 harness 自身速度
   ├─ same model
   ├─ same endpoint
   ├─ same reasoning / token config
   ├─ same tasks / environment
   └─ different harness
```

最终展示必须同时保留两类结果，避免把模型配置收益误归因给 harness。

## 10. 当前非目标

```text
多租户安全       容器编排       hardened sandbox
privilege split  capability     workflow/DAG engine
大型内置工具集   内置推理引擎   GPU/ANE/CUDA 调度
分布式 worker    为生态而做的复杂插件层
```

只有 Terminal-Bench 收益或核心运行需求成立后才重新评估。

## 11. 尚未锁定

| 主题 | 当前状态 |
|---|---|
| Terminal-Bench 2.1 Harbor adapter | P0 待实现 |
| 首批对照 Agent | Pi 优先，其他待定 |
| accuracy floor | 待 baseline 后确定 |
| reasoning effort 最优点 | 待 sweep |
| output token budget | 待 sweep |
| Provider 最小接口 | 待实现验证 |
| OpenAI-compatible 是否作为 baseline | 待验证 |
| streaming 细节 | 待验证 |
| connection/session persistence | 待 benchmark |
| `spawn` vs persistent shell | 待 benchmark |
| session 持久化 | 尚无必要性 |
| 插件/扩展机制 | 尚无必要性 |
| CLI/TUI | benchmark 需要决定 |
| 打包/分发 | 延后 |

原则：**让 Terminal-Bench 轨迹和 Harbor timing 决定优化方向。**

## 12. 文档规则

项目文档统一使用简体中文，并遵循 [`文档规范.md`](./文档规范.md)。

## 13. 开发花絮

非正式历史、命名过程、弃案与内部梗见 [`花絮/`](./花絮/README.md)。

这些内容用于保留项目演化过程，不作为当前设计约束。
