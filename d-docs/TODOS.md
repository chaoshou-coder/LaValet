# La Valet（V）TODO

优先级：`P0 = 跑通 Terminal-Bench` ｜ `P1 = 拉开 E2E 速度差` ｜ `P2 = 只优化已测瓶颈`

## 总览

```text
P0  Terminal-Bench 2.1 跑起来
 │
 ▼
P1  与 Pi 拉开真实任务 E2E 差距
 │
 ▼
P2  沿 Harbor timing / trajectory 继续压缩
```

不再调研或筛选其他 benchmark。评测协议见 [`BENCHMARK.md`](./BENCHMARK.md)。

## P0｜最小 benchmark 闭环

### Harbor / Terminal-Bench
- [ ] 实现 V 的 Harbor Custom Agent adapter
- [ ] 跑通 `terminal-bench/terminal-bench-2-1` 单任务
- [ ] 跑通多任务 job
- [ ] 保存 Harbor `result.json` / trajectory / token 数据
- [ ] 提取 `agent_execution` duration
- [ ] 自动汇总 reward / accuracy / p50 / p95 / p99
- [ ] 固定 benchmark 环境、并发度、provider 配置
- [ ] 建立 Pi 同任务 baseline

### Harness
- [ ] 初始化 Bun + TypeScript 项目
- [ ] 定义最小 message/session 类型
- [ ] 实现 Agent loop：`input → model → tool → observation → model → final`
- [ ] 增加最小 CLI / Harbor 入口
- [ ] Linux/macOS 跑通同一核心路径

### Provider
- [ ] 定义 harness 真正需要的最小 provider contract
- [ ] 接入首个 Terminal-Bench 使用的模型 API
- [ ] 支持 provider 原生 streaming
- [ ] 支持 reasoning effort / output token budget 配置
- [ ] 记录 model / effort / token limit / provider
- [ ] local/cloud 保持同一概念边界，不提前泛化

### Tools

| 工具 | 最小职责 |
|---|---|
| `read` | 读文件 |
| `write` | 创建/覆盖文件 |
| `edit` | 精确修改文件 |
| `bash` | `Bun.spawn()` 直接执行 |

- [ ] 完成四工具实现
- [ ] 解析 model tool call
- [ ] observation 回注模型
- [ ] 支持多轮 tool loop

### Token / Thinking
- [ ] baseline：默认 reasoning / token 配置
- [ ] sweep：`off/minimal/low/...`（按 provider 实际支持项）
- [ ] sweep：visible output token budget
- [ ] tool-first：删除工具调用前无价值解释
- [ ] final-only prose：中间轮尽量只产生 tool call
- [ ] 记录 reasoning/output tokens（provider 可提供时）
- [ ] 找出 accuracy ↔ latency Pareto frontier
- [ ] 暂不实现复杂 adaptive reasoning，先证明静态配置收益

### 极简护栏
- [ ] 保持单进程
- [ ] session 仅放内存
- [ ] 不加 database
- [ ] 不加 sandbox / privilege split
- [ ] 不加 worker pool / scheduler / DAG
- [ ] 不加 plugin framework
- [ ] 不为 Linux/macOS 预造 OS abstraction
- [ ] 不为 benchmark 权威性增加额外准备工作
- [ ] 生产价值不能单独构成新增组件的理由

## P1｜拉开 E2E 速度差

### Rank Track

目标：完整 V 系统在 Terminal-Bench 2.1 上尽可能快。

```text
优先级
1. 少 model round trip
2. 少 reasoning token
3. 少 visible output token
4. 少 context / observation token
5. 并行 / batching / early dispatch
6. provider / shell / harness hot path
```

- [ ] 选择 accuracy floor
- [ ] reasoning effort sweep
- [ ] output token sweep
- [ ] prompt / tool schema 压缩
- [ ] context / observation 压缩
- [ ] model turns / tool calls 统计
- [ ] 尝试减少无效 model turn
- [ ] 找 Pareto 最优配置
- [ ] 保存完整配置与 commit SHA

### Harness Track

目标：解释 V 的速度优势中有多少来自 harness 本身。

固定：

```text
model / endpoint / reasoning effort / token budget
sampling / tasks / environment / concurrency
```

- [ ] 尽量统一 V / Pi tool capability
- [ ] 同一模型配置跑 V / Pi
- [ ] 比较 `agent_execution` p50 / p95 / p99
- [ ] 比较 TTFA / ITL / turns / tool calls / tokens
- [ ] 记录无法完全控制的差异

### Tracing

```text
input
 │ T0
 ▼
request prepare
 │ T1
 ▼
provider dispatch
 │ T2
 ▼
first byte/token
 │ T3
 ▼
tool args complete
 │ T4
 ▼
tool dispatch
 │ T5
 ▼
tool complete
 │ T6
 ▼
observation ready
 │ T7
 ▼
next provider dispatch
```

- [ ] TTFA
- [ ] ITL
- [ ] T_E2E
- [ ] p50 / p95 / p99
- [ ] 区分 harness overhead 与 provider/model latency
- [ ] 对齐 Harbor `agent_execution`

### Tool hot path

| 项目 | 要测什么 |
|---|---|
| `Bun.spawn()` | 启动固定成本 |
| 短命令 | `true/pwd/rg/git status` |
| 文件工具 | 独立于模型的执行开销 |
| observation | 小/大结果序列化成本 |
| shell 模型 | spawn-per-call vs persistent shell |
| 中断 | timeout/cancel/signal 成本与正确性 |

### Provider hot path
- [ ] 连接建立成本
- [ ] keep-alive 是否消除重复连接成本
- [ ] 历史增长时 request serialization 成本
- [ ] provider 是否支持 prefix/KV/session reuse
- [ ] 区分通用优化与 provider-specific 优化

### 首个 baseline 记录

| 指标 | 记录 |
|---|---|
| Terminal-Bench accuracy | [ ] |
| `agent_execution` p50/p95/p99 | [ ] |
| 与 Pi 的 speedup | [ ] |
| model turns | [ ] |
| tool calls | [ ] |
| prompt / cached / reasoning / output tokens | [ ] |
| SLOC | [ ] |
| dependency count | [ ] |
| cold start | [ ] |
| idle memory | [ ] |
| mock provider 每轮 harness overhead | [ ] |

> 所有优化前保存 baseline，避免失去归因。

## P2｜只有测出瓶颈才做

| 候选优化 | 进入条件 |
|---|---|
| adaptive reasoning | 静态低 effort 明显丢 accuracy，且高 effort 明显增延迟 |
| persistent shell | spawn 明显占 ITL |
| persistent provider/session transport | 连接/协议成本显著 |
| delta conversation transport | provider 真正支持且历史重传显著 |
| streamed args early dispatch | 参数等待时间可测 |
| read-only 并行执行 | Terminal-Bench 轨迹存在并行机会 |
| batched/compound tools | round-trip amplification 足够大 |
| UDS local provider | 当前 transport 成为瓶颈 |
| binary/custom protocol | serialization/protocol 成本被证明 |
| Linux/macOS fast path | Terminal-Bench 收益显著 |

## 明确延后

```text
其他 benchmark landscape / 权威性比较
sandbox / container orchestration / capability system
multi-tenant security / generic workflow engine
大型 native tool catalog / built-in inference runtime
GPU/Metal/CUDA/ANE scheduling / distributed workers
为“生态完整”而预造的 MCP/plugin 层
```

## P2 晋级检查

任何候选进入核心前必须回答：

1. 能降低 Terminal-Bench E2E 多少？
2. accuracy 是否仍在可接受范围？
3. 增加多少概念、依赖、进程、状态、分支？
4. Bun / OS / provider / Unix 工具是否已经解决？
5. 能否留在核心之外？

```text
无明显 E2E 收益 → 不做
收益强 + 复杂度低 → 做
收益强 + 复杂度高 → 继续找更简单方案
```
