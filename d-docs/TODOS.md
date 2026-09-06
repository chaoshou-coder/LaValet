# La Valet（V）TODO

优先级：`P0 = 最小可用闭环` ｜ `P1 = 可测量、可验证` ｜ `P2 = 仅在证据成立后优化`

## 总览

```text
P0  跑起来
 │
 ▼
P1  测清楚
 │
 ▼
P2  只优化真实瓶颈
```

## P0｜最小产品闭环

### Harness
- [ ] 初始化 Bun + TypeScript 项目
- [ ] 定义最小 message/session 类型
- [ ] 实现 Agent loop：`input → model → tool → observation → model → final`
- [ ] 增加最小 CLI 入口
- [ ] Linux/macOS 跑通同一核心路径

### Provider
- [ ] 定义 harness 真正需要的最小 provider contract
- [ ] 接入 1 个云端 LLM API
- [ ] 接入 1 个本地 LLM server API
- [ ] 支持 provider 原生 streaming
- [ ] 验证 OpenAI-compatible 语义能否作为首版 baseline
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

### 极简护栏
- [ ] 保持单进程
- [ ] session 仅放内存
- [ ] 不加 database
- [ ] 不加 sandbox / privilege split
- [ ] 不加 worker pool / scheduler / DAG
- [ ] 不加 plugin framework
- [ ] 不为 Linux/macOS 预造 OS abstraction
- [ ] 大组件进入核心前必须有需求或 benchmark 证据

## P1｜测量与验证

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
- [ ] 固定一组多轮 benchmark tasks

### Linux + macOS 交集
- [ ] 枚举 baseline 使用的 Bun/OS 原语
- [ ] 验证每个原语在 Linux/macOS 行为一致性
- [ ] Linux smoke test / CI
- [ ] macOS smoke test / CI
- [ ] 记录不可避免的平台语义差异

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
- [ ] 云端连接建立成本
- [ ] 本地 server 连接建立成本
- [ ] keep-alive 是否已消除重复连接成本
- [ ] 历史增长时 request serialization 成本
- [ ] provider 是否支持 prefix/KV/session reuse
- [ ] 区分通用优化与 provider-specific 优化

### Correctness
- [ ] `read` 路径/范围
- [ ] `write` 创建/覆盖
- [ ] `edit` 匹配/失败
- [ ] `bash` stdout/stderr/exit code
- [ ] malformed tool call
- [ ] model stream 中断
- [ ] tool 执行失败
- [ ] deterministic mock provider 多轮测试
- [ ] Linux/macOS signal 与 subprocess termination

### 首个 baseline 记录

| 指标 | 记录 |
|---|---|
| SLOC | [ ] |
| dependency count | [ ] |
| cold start | [ ] |
| idle memory | [ ] |
| mock provider 每轮 harness overhead | [ ] |
| local provider T_E2E | [ ] |
| cloud provider T_E2E | [ ] |

> 所有优化前先保存 baseline，避免“优化后无对照”。

## P2｜只有测出瓶颈才做

| 候选优化 | 进入条件 |
|---|---|
| persistent shell | spawn 明显占 ITL |
| persistent provider/session transport | 连接/协议成本显著 |
| delta conversation transport | provider 真正支持且历史重传显著 |
| streamed args early dispatch | 参数等待时间可测 |
| read-only 并行执行 | 真实轨迹存在并行机会 |
| batched/compound FS tools | round-trip amplification 足够大 |
| UDS local provider | 当前 transport 成为瓶颈 |
| binary/custom protocol | serialization/protocol 成本被证明 |
| Linux/macOS fast path | 收益 > 维护成本 |

## 明确延后

```text
sandbox / container orchestration / capability system
multi-tenant security / generic workflow engine
大型 native tool catalog / built-in inference runtime
GPU/Metal/CUDA/ANE scheduling / distributed workers
为“生态完整”而预造的 MCP/plugin 层
```

## P2 晋级检查

任何候选进入核心前必须回答：

1. 解决了哪个**已测量瓶颈**或**必需能力**？
2. 收益是多少？
3. 增加多少概念、依赖、进程、状态、分支？
4. Bun / OS / provider / Unix 工具是否已经解决？
5. 能否留在核心之外？

```text
证据弱 → 不做
证据强 + 复杂度低 → 做
证据强 + 复杂度高 → 继续寻找更简单方案
```
