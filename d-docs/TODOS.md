# La Valet（V）TODO

当前状态：**S1｜最小可运行 Agent**

状态定义和验收门槛见 [`STATUS.md`](./STATUS.md)。本文件只记录当前状态的工作；进入下一状态时整体替换。

## 目标

得到一个足够小、功能正确的 V。当前不做 Harbor benchmark、Pi baseline 或性能优化。

## 当前工作

### Harness

- [ ] 初始化 Bun + TypeScript 项目
- [ ] 定义最小 message / session 数据结构
- [ ] 实现 Agent loop：`input → model → tool → observation → model → final`
- [ ] 增加最小 CLI 入口

### Provider

- [ ] 定义当前 Agent loop 真正需要的最小 Provider contract
- [ ] 接入首个真实模型 API
- [ ] 支持 provider 原生 streaming
- [ ] local / cloud 保持同一 Provider 概念边界，不提前泛化

### Tools

| 工具 | 最小职责 |
|---|---|
| `read` | 读文件 |
| `write` | 创建 / 覆盖文件 |
| `edit` | 精确修改文件 |
| `bash` | `Bun.spawn()` 直接执行 |

- [ ] 四工具可独立正确执行
- [ ] 解析 model tool call
- [ ] observation 回注模型
- [ ] 支持连续多轮 tool loop

### 边界检查

- [ ] session 仅使用进程内存
- [ ] 核心执行路径保持单进程
- [ ] Linux / macOS 使用同一套核心实现
- [ ] 两个平台完成 smoke test
- [ ] 不加入 S1 验收不需要的 framework / daemon / database / sandbox / plugin system

## S1 完成条件

以下条件全部满足后，将项目状态切换到 **S2｜Benchmark 闭环**，并重新生成本文件：

- Bun + TypeScript 项目可运行
- Provider 完成真实 streaming 模型调用
- 四工具独立可用
- tool call → execution → observation 链路正确
- 多轮 Agent loop 可以正常结束
- 单进程 + 内存 session 边界成立
- Linux / macOS 同核心路径 smoke test 通过
