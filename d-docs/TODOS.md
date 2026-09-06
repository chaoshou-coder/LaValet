# La Valet — TODOs

This file tracks the next work needed to turn the current project decisions into a minimal, measurable baseline.

Priority convention:

- **P0** — required for the first useful closed loop;
- **P1** — required to evaluate the architecture properly;
- **P2** — optional optimization or expansion after evidence exists.

## P0 — Minimal product loop

- [ ] Initialize the Bun + TypeScript project.
- [ ] Define the smallest possible internal message/session types.
- [ ] Implement one provider adapter sufficient to call an LLM API.
- [ ] Support streaming model output where the chosen provider exposes it.
- [ ] Parse model tool calls into the V tool loop.
- [ ] Implement `read`.
- [ ] Implement `write`.
- [ ] Implement `edit`.
- [ ] Implement `bash` using direct Bun process spawning.
- [ ] Feed tool observations back to the model.
- [ ] Complete multi-round model ↔ tool execution until final response.
- [ ] Add a minimal CLI entry point.
- [ ] Verify the same core path on both Linux and macOS.

## P0 — Provider boundary

- [ ] Write down the minimum provider contract required by the harness.
- [ ] Keep local and cloud models behind the same conceptual provider boundary.
- [ ] Validate the provider contract against at least one cloud API.
- [ ] Validate the same contract against at least one local LLM server API.
- [ ] Decide whether OpenAI-compatible request/response semantics are sufficient for the first implementation.
- [ ] Avoid provider-generalization beyond what those first concrete integrations require.

## P0 — Simplicity guardrails

- [ ] Keep the first implementation single-process unless a requirement makes that impossible.
- [ ] Keep session state in memory.
- [ ] Do not add a database.
- [ ] Do not add a sandbox.
- [ ] Do not add privilege separation.
- [ ] Do not add a worker pool.
- [ ] Do not add a scheduler/DAG engine.
- [ ] Do not add a plugin framework before extension pressure exists.
- [ ] Do not add an OS abstraction layer while Bun already covers the needed Linux/macOS intersection.
- [ ] Require a concrete requirement or benchmark result before adding a major architectural component.

## P1 — Benchmark and tracing foundation

- [ ] Define a repeatable end-to-end benchmark harness.
- [ ] Instrument **TTFA** (Time To First Action).
- [ ] Instrument **ITL** (Inter-Tool Latency).
- [ ] Instrument **T_E2E** (full task completion latency).
- [ ] Record p50 / p95 / p99 where enough samples exist.
- [ ] Timestamp model request preparation.
- [ ] Timestamp request serialization / dispatch.
- [ ] Timestamp first model byte/token.
- [ ] Timestamp completed tool-call arguments.
- [ ] Timestamp tool dispatch.
- [ ] Timestamp tool completion.
- [ ] Timestamp observation preparation.
- [ ] Timestamp next-round model dispatch.
- [ ] Separate harness overhead from provider/model latency.
- [ ] Create a small fixed benchmark task set that requires multiple tool rounds.

## P1 — Linux + macOS compatibility

- [ ] Enumerate every OS/runtime primitive used by the baseline.
- [ ] Confirm each primitive works through Bun on both Linux and macOS.
- [ ] Add CI or equivalent automated smoke tests for Linux.
- [ ] Add CI or equivalent automated smoke tests for macOS.
- [ ] Treat platform-specific code as an exception that requires justification.
- [ ] Document any unavoidable semantic difference between Linux and macOS.

## P1 — Tool-path measurements

- [ ] Benchmark `Bun.spawn()` startup overhead on Linux and macOS.
- [ ] Benchmark representative short commands such as `true`, `pwd`, `rg`, and `git status`.
- [ ] Compare spawn-per-command against a persistent shell only after the baseline exists.
- [ ] Measure filesystem-tool overhead independently from model latency.
- [ ] Measure tool-result serialization overhead for small and large observations.
- [ ] Test cancellation and timeout behavior without introducing a large execution framework.

## P1 — Provider-path measurements

- [ ] Measure connection setup cost for cloud APIs.
- [ ] Measure connection setup cost for local model servers.
- [ ] Verify whether HTTP keep-alive / persistent connections eliminate meaningful repeated setup cost.
- [ ] Measure request serialization cost with growing conversation history.
- [ ] Investigate provider-side prefix/KV/session reuse where supported.
- [ ] Determine what can be optimized generically in V versus what belongs to a specific provider adapter.

## P1 — Correctness tests

- [ ] Unit-test `read` path/range behavior.
- [ ] Unit-test `write` replacement/creation behavior.
- [ ] Unit-test `edit` matching and failure cases.
- [ ] Unit-test `bash` stdout/stderr/exit-code handling.
- [ ] Test malformed tool calls.
- [ ] Test interrupted model streams.
- [ ] Test failed tool execution.
- [ ] Test multi-round loops with deterministic mock provider responses.
- [ ] Test Linux/macOS behavior for signals and subprocess termination.

## P1 — Define the first benchmark baseline

- [ ] Record source line count / dependency count for the baseline harness.
- [ ] Record cold-start time.
- [ ] Record idle memory footprint.
- [ ] Record per-round harness overhead with a mock model provider.
- [ ] Record real local-provider E2E latency.
- [ ] Record real cloud-provider E2E latency.
- [ ] Preserve baseline numbers before adding optimizations.

## P2 — Only after measurement

- [ ] Evaluate persistent shell execution if process spawn is a material ITL contributor.
- [ ] Evaluate a persistent provider/session transport if connection/protocol overhead is material.
- [ ] Evaluate incremental/delta conversation transport where a provider can actually consume it.
- [ ] Evaluate early tool dispatch from streamed arguments if parsing delay is measurable.
- [ ] Evaluate parallel read-only tool execution where task traces justify it.
- [ ] Evaluate compound/batched filesystem operations only if round-trip amplification justifies the added API surface.
- [ ] Evaluate Unix domain sockets for local providers only if the existing transport is a measurable bottleneck.
- [ ] Evaluate binary/custom protocols only after serialization/protocol overhead is demonstrated.
- [ ] Evaluate platform-specific Linux or macOS fast paths only when their gain outweighs maintenance cost.

## Deferred / explicitly out of scope

Do not implement these merely because mature Agent frameworks have them:

- [ ] sandbox framework;
- [ ] container orchestration;
- [ ] privilege/capability subsystem;
- [ ] multi-tenant security model;
- [ ] generic workflow engine;
- [ ] large native tool catalogue;
- [ ] built-in model inference runtime;
- [ ] GPU/Metal/CUDA/ANE scheduling inside the harness;
- [ ] remote distributed worker architecture;
- [ ] generalized MCP-style ecosystem layer before a concrete need exists.

## Decision checkpoints

Before promoting any P2 item into the core, answer:

1. What measured bottleneck or required product capability does this solve?
2. How much latency/capability does it improve?
3. How many concepts, dependencies, processes, states, or branches does it add?
4. Can Bun, the OS, the provider, or an existing Unix tool already solve it?
5. Can the feature remain optional and outside the core?

If these questions do not produce a strong justification, keep the feature out of V.
