# Le Valet（V）项目决策

> 旧代号：`delta-agent` ｜ 中文名：拉挽乐 ｜ 仓库：`chaoshou-coder/LaValet`

## 1. 首要目标

**V 的首要目标是提高面试竞争力：在真实 Agent benchmark 中做出显著、可复现、可解释的速度优势。**

```text
面试竞争力
   │
   ▼
可展示的速度优势
   │
   ▼
Terminal-Bench 2.1 + Harbor
   │
   ├─ reward / accuracy：任务确实完成
   └─ agent_execution：真实 E2E latency
```

开发优先级：

```text
Terminal-Bench E2E 速度优势
            ↓
可复现 / 可解释 / 可归因
            ↓
极简
            ↓
benchmark 权威性 / 生产通用性 / 安全 / 生态
```

Terminal-Bench 2.1 已固定为主战场。不再为了寻找“更权威”或“理论上更合适”的 benchmark 增加前置调研；除非它实际阻碍速度优势展示，否则不切换。

详细评测协议见 [`BENCHMARK.md`](./BENCHMARK.md)。

## 2. 一句话定义

**V 是一个面向 Terminal-Bench 2.1 真实任务 E2E 速度优化、运行于 Linux + macOS 公共交集上的极简 Agent Harness。**

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
| Benchmark | Terminal-Bench 2.1 + Harbor | 固定主战场 |
| 首要目标 | 最大化真实任务 E2E 速度优势 | 优先优化 `agent_execution` |
| 能力约束 | reward / accuracy | 防止快速失败污染结果 |
| 首要对照 | Pi | 优先建立同任务速度差 |
| 平台 | Linux + macOS 公共交集 | 不做 macOS 专属核心 |
| Runtime | Bun | 优先直接使用 Bun 原语 |
| Inference | 外部 API | 不内置推理 runtime |
| Provider | 本地 / 云端统一看待 | local LLM 无特殊架构地位 |
| 核心工具 | `read/write/edit/bash` | 默认不增加第 5 个工具 |
| 安全 | 非核心目标 | 可为简单性与 benchmark 收益让位 |
| 核心原则 | speed-first + 极简 | 新机制必须证明 E2E 收益 |

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
Terminal-Bench E2E 实测 > 假想需求
```

约束：

1. Bun 已解决的平台差异，不再包一层 platform abstraction。
2. 新组件优先由 Terminal-Bench / Harbor 数据证明必要性。
3. 不为了 benchmark 权威性增加准备工作或切换任务集。
4. 不因为成熟 Agent 框架“都有”某功能就引入它。
5. 能显著降低 E2E 且保持可接受 accuracy 的优化优先。

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

模型侧参数可作为 **Rank Track** 的速度变量，包括 reasoning effort、token budget 等；**Harness Track** 锁定这些参数做公平归因。

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

优化优先级首先看完整 E2E，而不是纯 harness 微秒/毫秒开销。因此减少 reasoning/output token、model turn 和 tool round-trip 属于核心性能工作。

## 9. Benchmark 双赛道

```text
Rank Track
└─ 追求完整系统最低 E2E
   ├─ reasoning / token budget
   ├─ prompt / policy
   ├─ turns / context / tools
   └─ harness / provider / transport

Harness Track
└─ 解释为什么快
   ├─ same model
   ├─ same endpoint
   ├─ same reasoning / token config
   ├─ same tasks / environment
   └─ different harness
```

Rank Track 是主目标；Harness Track 服务于速度归因。

## 10. 当前非目标

```text
其他 benchmark 横向调研 / benchmark 权威性优化
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
| 首批对照 Agent | Pi 优先 |
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

原则：**先跑 Terminal-Bench baseline，再让 Harbor timing 和 trajectory 决定优化方向。**

## 12. 文档规则

项目文档统一使用简体中文，并遵循 [`文档规范.md`](./文档规范.md)。

## 13. 开发花絮

非正式历史、命名过程、弃案与内部梗见 [`花絮/`](./花絮/README.md)。

这些内容用于保留项目演化过程，不作为当前设计约束。
