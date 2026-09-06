# La Valet（V）项目决策

> 旧代号：`delta-agent` ｜ 中文名：拉挽乐 ｜ 仓库：`chaoshou-coder/LaValet`

## 1. 一句话定义

**V 是一个运行于 Linux + macOS 公共交集上的极简 Agent Harness。**

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

## 2. 已锁定边界

| 项目 | 决策 | 含义 |
|---|---|---|
| 平台 | Linux + macOS 公共交集 | 不做 macOS 专属核心 |
| Runtime | Bun | 优先直接使用 Bun 原语 |
| Inference | 外部 API | 不内置推理 runtime |
| Provider | 本地 / 云端统一看待 | local LLM 无特殊架构地位 |
| 核心工具 | `read/write/edit/bash` | 默认不增加第 5 个工具 |
| 安全 | 非核心目标 | 可为简单性放弃 sandbox / privsep |
| 核心原则 | 极简优先 | 新机制必须证明自身必要性 |

## 3. 最小执行模型

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

## 4. 极简规则

```text
函数           > class hierarchy
单进程         > IPC
Bun / OS 原语  > framework
现有 Unix 工具 > harness 内重写
内存状态       > database
现有协议       > 自定义协议
实测瓶颈       > 假想瓶颈
```

约束：

1. Bun 已解决的平台差异，不再包一层 platform abstraction。
2. 只有真实需求或 benchmark 能证明新组件存在的必要性。
3. 平台专属 fast path 只能是可选优化，不能污染核心路径。
4. 不因为成熟 Agent 框架“都有”某功能就引入它。

## 5. Inference 边界

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

## 6. 平台边界

V 使用 **Linux 与 macOS 的实际公共能力子集**，不假设二者互为超集。

优先级：

```text
Bun 跨平台 API
      ↓
Unix/POSIX 公共原语
      ↓
平台专属能力（仅在收益被证明后）
```

目标不是“抽象所有 OS 差异”，而是尽量让差异根本不进入核心。

## 7. 性能目标

核心指标是完整任务延迟：

| 指标 | 含义 |
|---|---|
| TTFA | 用户输入 → 第一个有效动作 |
| ITL | 相邻工具动作之间的延迟 |
| T_E2E | 输入 → 任务真正完成 |
| p50/p95/p99 | 延迟分布 |

```text
T_E2E = harness + provider + model + tool + observation + ... + final
```

多轮任务会放大每轮固定开销，因此优化依据必须来自 critical path tracing。

## 8. 当前非目标

```text
多租户安全       容器编排       hardened sandbox
privilege split  capability     workflow/DAG engine
大型内置工具集   内置推理引擎   GPU/ANE/CUDA 调度
分布式 worker    为生态而做的复杂插件层
```

以后可以重新评估，但不能预先进入核心。

## 9. 尚未锁定

| 主题 | 当前状态 |
|---|---|
| Provider 最小接口 | 待实现验证 |
| OpenAI-compatible 是否作为 baseline | 待验证 |
| streaming 细节 | 待验证 |
| connection/session persistence | 待 benchmark |
| `spawn` vs persistent shell | 待 benchmark |
| session 持久化 | 尚无必要性 |
| 插件/扩展机制 | 尚无必要性 |
| CLI/TUI | 待设计 |
| 打包/分发 | 待设计 |
| Linux/macOS 专属 fast path | 待 benchmark |

原则：**让实现压力和测量结果决定架构，而不是提前设计框架。**

## 10. 文档规则

项目文档统一使用简体中文，并遵循 [`文档规范.md`](./文档规范.md)。
