# Le Valet（V）项目决策

> 旧代号：`delta-agent` ｜ 中文名：拉挽乐 ｜ 仓库：`chaoshou-coder/LaValet`

本文件记录项目级最终决策及其原因。Benchmark 细节以 [`BENCHMARK.md`](./BENCHMARK.md) 为唯一权威位置。

## 1. 立项目标

| 最终决策 | 原因 |
|---|---|
| V 的首要目标是提高面试竞争力 | 项目首先承担技术能力证明作用，不以解决生产痛点为第一目标 |
| 主要通过显著、可复现、可解释的 Agent E2E 速度优势展示能力 | Benchmark 数字能够直接展示系统优化、测量、归因和工程取舍能力 |
| 生产通用性、安全、生态完整度均低于 benchmark 速度目标 | 这些目标会引入大量与当前面试展示价值弱相关的复杂度和准备工作 |

优先级：

```text
面试竞争力
   ↓
真实任务 E2E 速度优势
   ↓
可复现 / 可解释 / 可归因
   ↓
极简
   ↓
生产通用性 / 安全 / 生态完整度
```

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

## 3. 已锁定技术边界

| 最终决策 | 原因 |
|---|---|
| 平台：Linux + macOS 公共交集 | 保持足够通用，同时避免为单一平台引入额外核心结构 |
| Runtime：Bun | 减少 Node 兼容层与依赖，优先直接使用 Bun 提供的高性能原语 |
| Inference：外部 API | V 的核心任务是 harness；推理 runtime、kernel、GPU 调度不应污染核心边界 |
| 本地 / 云端模型统一通过 Provider | local LLM 只是 provider 的一种，不需要特殊架构地位 |
| 核心工具固定为 `read / write / edit / bash` | 四个原语已能覆盖 coding Agent 的最小闭环，并保持工具面和 prompt schema 极小 |
| 单进程优先 | 避免 IPC、daemon、worker 带来的固定开销和状态复杂度 |
| session 默认仅放内存 | 当前 benchmark 场景不需要数据库或持久化系统 |
| 并发边界：不面向多用户；单用户内部允许多个 model request 并发；tool call 并发作为劣后 feature | 保留单用户场景所需的内部并行能力，同时不为多用户调度与非验收 feature 提前增加复杂度 |
| 安全隔离不是核心目标 | 单用户可信环境下，sandbox / privsep 的收益低于其复杂度与延迟成本 |
| 不预造 plugin / workflow / capability framework | 这些机制只有在 benchmark 或核心运行需求证明必要时才进入项目 |

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

Baseline：

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

## 5. 极简原则

| 最终决策 | 原因 |
|---|---|
| 函数优先于 class hierarchy | 降低抽象和调用层级 |
| 单进程优先于 IPC | 降低固定通信成本 |
| Bun / OS 原语优先于 framework | 避免重复封装已有能力 |
| 现有 Unix 工具优先于 harness 内重写 | 减少代码量和维护面 |
| 内存状态优先于 database | 当前任务不需要持久状态系统 |
| 现有协议优先于自定义协议 | 只有测出协议瓶颈后才值得承担自定义成本 |
| 实测 E2E 瓶颈优先于假想需求 | 项目面向 benchmark 迭代，不能被未验证的生产需求牵引 |

## 6. Inference 边界

```text
                   ┌─ OpenAI / Anthropic / ...
V Provider Layer ──┤
                   ├─ llama.cpp server
                   ├─ BaseRT server
                   └─ 其它兼容 API
```

V 核心不依赖：

```text
CUDA / Metal / ANE / MLX / llama.cpp / BaseRT / vLLM / 特定模型
```

原因：这些属于 inference runtime 层；V 只优化 harness 到 provider 的 Agent 执行路径。

模型侧 reasoning / token 等参数仍可作为完整系统速度变量，具体规则见 [`BENCHMARK.md`](./BENCHMARK.md)。

## 7. Benchmark 决策

最终固定：

```text
Terminal-Bench 2.1 = 开发靶场
Harbor             = 测量工具
Pi                 = 首要对照
```

原因、指标、双赛道、thinking/token 策略和开发循环统一记录于 [`BENCHMARK.md`](./BENCHMARK.md)，本文件不重复维护。

## 8. 当前非目标

```text
其他 benchmark 横向调研
多用户并发 / 多租户安全
容器编排 / hardened sandbox
privilege split / capability system
workflow / DAG engine
大型内置工具集
内置推理引擎
GPU / Metal / CUDA / ANE 调度
分布式 worker
为生态完整度预造的 MCP / plugin 层
```

原因：当前没有证据表明这些工作能提高 Terminal-Bench 2.1 的 E2E 速度展示价值。

## 9. 尚未锁定

以下问题必须通过实现和 benchmark 决定：

```text
Provider 最小接口
OpenAI-compatible baseline
streaming 细节
connection / session persistence
spawn vs persistent shell
reasoning effort 最优点
output token budget
accuracy floor
batching / early dispatch
observation compression
```

原则：**先跑 baseline，再让 Harbor timing 和 trajectory 决定架构。**

## 10. 文档规则

项目文档正文统一使用简体中文；所有文件夹名和文件名必须使用 ASCII 安全字符。完整规范见 [`DOCS_STYLE.md`](./DOCS_STYLE.md)。

开发花絮、命名过程和弃案记录于 [`lore/`](./lore/README.md)，不作为当前设计约束。
