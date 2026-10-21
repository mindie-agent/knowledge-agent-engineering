---
schema: mindie-graded-unit/1
id: unit-cann-model-infer-optimize-role-isolation-and-progress-relay-workflow
revision: 1
title: "CANN Model-Infer-Optimize Multi-Agent Workflow: Seven-Stage Pipeline, Four-Role Isolation, and Two-Zone progress.md Relay"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: orchestration/cann-recipes-infer/model-infer-optimize/stage-gated-role-isolation-and-progress-relay
inherits_from:
  - unit-harness-workflow-dag-state-machine-and-approval-gate-workflow
  - unit-superpowers-claim-evidence-scope
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: orchestration
triggers:
  error_signatures:
    - "model-infer-optimize"
    - "progress_history.md"
    - "baseline_metadata.json"
  keywords:
    - "model-infer-optimize"
    - "role-isolation"
    - "progress-md-relay"
    - "analyzer-implementer-reviewer"
    - "cann-recipes-infer"
provenance:
  source_type: source_contract
  source_locator: cann/cann-recipes-infer@3c9266727c2a25bd17d503ad37d189a134ff364b:docs/agent/model-infer-optimize-design.md
  evidence_refs:
    - https://github.com/cann/cann-recipes-infer/blob/3c9266727c2a25bd17d503ad37d189a134ff364b/docs/agent/model-infer-optimize-design.md#L1-L250
  supersedes: []
---

## Falsifiable Claim

In end-to-end NPU model inference adaptation (`model-infer-optimize`), allowing a single monolithic agent to analyze, edit model code, and judge its own accuracy/performance causes context window pollution and unverified code drift. Enforcing strict four-role authority isolation (`Main Agent`, `analyzer`, `implementer`, `reviewer`) across a seven-stage gated pipeline (`Stage 0` baseline $\rightarrow$ `Stage 6` summary) with a two-zone `progress.md` relay file ensures that every optimization stage is user-approved before implementation and independently verified against `baseline_metadata.json` before advancing.

## Four-Role Authority & Skill Isolation Matrix

| Role | Responsibility | File-System Authority | Mounted Domain Skills |
| :--- | :--- | :--- | :--- |
| **Main Agent** | Global stage progression, user confirmation gates, self-verification completeness checks, and `progress.md` archiving | **Forbidden from directly modifying model source code**; updates `progress.md` permanent overview table | `model-infer-optimize` |
| **`analyzer`** | Architecture inspection and stage optimization plan design | **Read-only on model code**; writes analysis proposals only to `progress.md` workspace | `parallel-analysis`, `kvcache`, `fusion`, `quantization`, `graph-mode` |
| **`implementer`** | Code adaptation, operator replacement, and mandatory self-verification | Read/write access to model source code and `progress.md` workspace | `migrator`, `parallel-impl`, `kvcache`, `fusion`, `quantization`, `graph-mode`, `precision-debug`, `runtime-debug` |
| **`reviewer`** | Independent numerical accuracy and latency/throughput verification against `baseline_metadata.json` | **Strictly forbidden from modifying model source code** | `precision-debug`, `runtime-debug` |

## Seven-Stage Gated Optimization Pipeline

1. **Stage 0 — Model Analysis & Baseline Establishment**:
   - `analyzer` inspects model architecture and hardware environment; `implementer` runs `model-infer-migrator` to complete framework adaptation and record `baseline/baseline_metadata.json`.
2. **Stage 1 — Multi-Card Parallelism (`model-infer-parallel-analysis` $\rightarrow$ `model-infer-parallel-impl`)**:
   - Recommends TP/DP/EP partitioning, implements distributed collectives, and verifies multi-card numerical parity via `reviewer`.
3. **Stage 2 — KVCache & FlashAttention Replacement (`model-infer-kvcache`)**:
   - Replaces eager attention and KV storage with Ascend FIA/PA operators; if `reviewer` detects accuracy regression, enters a repair loop with `implementer` loading `model-infer-precision-debug`.
4. **Stage 3 — Fused Operator Replacement (`model-infer-fusion`)**:
   - Matches subgraph patterns (`RMSNorm`, `RoPE`, `SwiGLU`, `MatMul+Add`) and replaces them with `aclnn` / ATB fused kernels.
5. **Stage 4 — Quantization Adaptation (`model-infer-quantization`, Optional)**:
   - Integrates W8A8 / W4A16 quantization when enabled during Stage 0 evaluation.
6. **Stage 5 — Graph Mode Adaptation (`model-infer-graph-mode`)**:
   - Adapts the model to TorchAir (`npugraph_ex` or GE mode), eliminating graph breaks and verifying static/dynamic bucketing.
7. **Stage 6 — Optimization Summary**:
   - Synthesizes cumulative latency/throughput gains and accuracy verification across all stages.
   - *(Cross-Stage Interrupt)*: `model-infer-runtime-debug` can be triggered at any stage upon AICore timeout, OOM, or HCCL hang.

## Two-Zone `progress.md` State Relay Protocol

To prevent context window overflow across stages and subagents, state is persisted in `{model_dir}/progress.md` with two strictly separated zones:
1. **Permanent Zone (Top)**: Written during Stage 0 with model architecture invariants and a stage-by-stage overview table updated only by the Main Agent. Never cleared.
2. **Active Workspace Zone (Bottom)**: Appended by `analyzer`, `implementer`, and `reviewer` during the current stage.
3. **Stage Transition Archiving**: When a stage passes `reviewer` verification and user confirmation, the Main Agent updates the Permanent Zone overview table, archives the Active Workspace content into `progress_history.md` (queried only via keyword search, never read in full), and clears the Active Workspace for the next stage.
4. **Concise Dispatch Rule**: Main Agent dispatch prompts specify only the working directory, task goal, and required Skill name—never pasting full plans or repeating `progress.md` contents in the prompt.
