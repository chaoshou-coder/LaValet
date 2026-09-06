# La Valet — Project Decisions

> This directory records project-level decisions, boundaries, and engineering direction for **La Valet**.

## 1. Project identity

- **Project name:** La Valet
- **Short name:** V
- **Former codename:** delta-agent
- **Chinese name:** 拉挽乐
- **Repository:** `chaoshou-coder/LaValet`

La Valet is an extremely small Agent Harness. Its core job is to connect an LLM to a minimal set of tools and drive the model ↔ tool loop with as little framework overhead and architectural machinery as possible.

## 2. Primary design objective

**Minimalism is the first-order constraint.**

V should prefer:

- fewer concepts;
- fewer layers;
- fewer processes;
- fewer dependencies;
- less persistent state;
- direct use of Bun / OS primitives;
- measurable end-to-end latency improvements over speculative abstractions.

Security is not a primary design objective. For the initial trusted local-user scenario, V may deliberately give up sandboxing, privilege separation, capability systems, and related security machinery when they increase complexity without serving the core product goal.

A new abstraction or component should have to justify its existence through a concrete product requirement or benchmark result.

## 3. Platform boundary — LOCKED

### Supported platform model

**Linux + macOS common subset.**

V is not macOS-exclusive.

The core should depend only on facilities that can be expressed cleanly through Bun and the practical Unix/POSIX-like intersection shared by Linux and macOS.

Platform-specific features may be added later as optional optimizations, but they must not become requirements of the core harness unless a future product decision explicitly changes this boundary.

### Consequence

Avoid creating a platform abstraction layer pre-emptively.

If Bun already provides a portable primitive, use it directly. Platform-specific modules should appear only when a real requirement or benchmark demonstrates that they are needed.

## 4. Runtime — LOCKED

**Runtime: Bun.**

V is designed around Bun rather than Node.js as its JavaScript/TypeScript runtime.

The core should prefer Bun-native primitives for:

- process spawning;
- filesystem access;
- streams;
- sockets / HTTP where appropriate;
- environment and process interaction.

Do not wrap Bun APIs in additional internal framework layers unless there is a demonstrated need.

## 5. Inference boundary — LOCKED

**Inference is external to the harness.**

V talks to models through APIs. The model implementation and inference runtime are not part of the core harness.

A provider may be:

- a local LLM server;
- a cloud LLM API;
- another compatible external inference service.

A local LLM is therefore **one API provider among others**, not a special architectural category.

V should not require CUDA, Metal, ANE, MLX, llama.cpp, BaseRT, vLLM, or any specific inference runtime in its core architecture.

Hardware-specific inference scheduling, kernels, GPU/ANE resource management, and model-runtime implementation are outside the current core boundary.

## 6. Core tool surface — CURRENT BASELINE

The intended minimal tool surface is:

1. `read`
2. `write`
3. `edit`
4. `bash`

The purpose of this constraint is to keep the model-facing tool vocabulary and harness implementation small.

`bash` deliberately exposes the existing Unix userland instead of reproducing a large catalogue of harness-native tools.

Do not add a fifth core tool without a concrete reason.

## 7. Execution model — CURRENT BASELINE

The simplest viable architecture is preferred:

```text
user input
    ↓
agent loop
    ↓
model API
    ↓
tool call
    ↓
read / write / edit / bash
    ↓
observation
    ↓
model API
    ↓
...
    ↓
final response
```

Current baseline assumptions:

- one Bun process where practical;
- in-memory session state where practical;
- direct process execution for shell work;
- no mandatory sandbox;
- no mandatory daemon architecture;
- no mandatory worker pool;
- no mandatory database;
- no mandatory plugin runtime;
- no mandatory DAG/task scheduler;
- no mandatory IPC layer between harness components.

These are baseline simplifications, not immutable requirements. Any added mechanism must justify itself.

## 8. Performance objective

The primary performance target is full task latency rather than isolated model throughput.

Important metrics include:

- **TTFA** — Time To First Action;
- **ITL** — Inter-Tool Latency;
- **T_E2E** — end-to-end task completion latency;
- latency distributions where useful: **p50 / p95 / p99**.

For multi-round Agent workloads, fixed overhead on every model ↔ tool round trip matters because it is amplified across the trajectory.

Optimization priority should therefore follow the measured critical path rather than optimize individual components in isolation.

## 9. Architectural rules

Current working rules:

- Use a function before inventing a class hierarchy.
- Use one process before introducing IPC.
- Use Bun / OS primitives before introducing a framework.
- Use existing Unix programs before reimplementing them inside the harness.
- Keep state in memory before introducing persistent storage.
- Avoid schedulers until scheduling is actually required.
- Avoid custom protocols until an existing protocol is measured as a bottleneck.
- Do not introduce cross-platform abstractions for differences that do not yet exist in the product.
- Do not optimize a hypothetical bottleneck; benchmark it first.

## 10. Explicit non-goals for the initial core

The initial core does not aim to provide:

- cloud multi-tenant isolation;
- container orchestration;
- Docker/Kubernetes integration as a requirement;
- a hardened sandbox;
- OpenBSD-style `pledge` / `unveil` equivalents;
- privilege separation;
- a full capability-security system;
- a built-in inference engine;
- hardware-specific model kernels;
- a large built-in tool ecosystem;
- a general workflow/DAG engine.

These features may be reconsidered only when the product has a demonstrated need for them.

## 11. Current unresolved decisions

The following are intentionally not locked yet:

- exact model-provider interface;
- streaming protocol details;
- whether the provider layer should normalize OpenAI-compatible APIs or expose a smaller V-native interface;
- persistent connection strategy per provider;
- shell execution strategy (`spawn` per call vs persistent shell) after benchmarking;
- session persistence format, if persistence is needed at all;
- plugin/extension mechanism, if one is needed;
- exact CLI/TUI surface;
- packaging and distribution strategy;
- whether any platform-specific fast paths are worth maintaining.

These should be decided from implementation pressure and benchmarks rather than from framework design in advance.
