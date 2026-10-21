---
schema: mindie-graded-unit/1
id: unit-cann-sota-approach-plan-loop-and-three-file-state-workflow
revision: 1
title: "CANN SOTA-Approach Exploratory Optimization: Multi-Source Candidate Discovery, Plan Self-Loop, and Three-File State Spindle"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: orchestration/cann-recipes-infer/sota-approach/plan-self-loop-and-three-file-state-spindle
inherits_from:
  - unit-harness-workflow-dag-state-machine-and-approval-gate-workflow
  - unit-stagnation-triggered-debiased-subcontext-exploration-workflow
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: orchestration
triggers:
  error_signatures:
    - "model-infer-sota-approach"
    - "plan-dashboard.md"
    - "model-infer-perf-breakdown"
  keywords:
    - "model-infer-sota-approach"
    - "plan-dashboard"
    - "plan-self-loop"
    - "perf-breakdown"
    - "candidate-discovery"
provenance:
  source_type: source_contract
  source_locator: cann/cann-recipes-infer@3c9266727c2a25bd17d503ad37d189a134ff364b:docs/agent/sota-approach-skill-design.md
  evidence_refs:
    - https://github.com/cann/cann-recipes-infer/blob/3c9266727c2a25bd17d503ad37d189a134ff364b/docs/agent/sota-approach-skill-design.md#L1-L430
  supersedes: []
---

## Falsifiable Claim

Once a model has a working accuracy-verified baseline, predetermined linear optimization pipelines miss model-specific bottlenecks and non-linear interactions across multi-stream, prefetch, SuperKernel, and fusion optimizations. The `model-infer-sota-approach` workflow replaces fixed stages with (1) baseline `round0` profiling analyzed via `model-infer-perf-breakdown`, (2) parallel multi-source `candidate` discovery (`multi-stream`, `wiki`, `perf-insight`), (3) a 3-state Plan self-loop (`待实现`, `通过`, `淘汰`) indexed by a global monotonic `roundN` counter with append-only Plan derivation, and (4) a three-file single-writer state architecture (`plan-dashboard.md`, `progress.md`, `plans/plan-<id>.md`) that survives context compression without state divergence.

## Eight-Step Exploratory Orchestration Architecture

### Steps 1–6: Linear Preparation & Parallel Candidate Discovery
1. **Scenario Confirmation (`Main Agent`)**: Inspects repository candidate scenarios, confirms target batch/sequence shapes and accuracy criteria with the user upfront so all downstream subagents execute non-interactively.
2. **Baseline Input & Accuracy Gate (`scenario` subagent)**: Constructs reproducible inputs and writes `scenario.md`.
3. **Baseline Profiling Capture (`profiling-instrumenter` subagent)**: Captures `round0` profiling (`kernel_details.csv`, timeline traces) via `model-infer-profiling`.
4. **Baseline Breakdown Analysis (`profile-analyzer` subagent)**: Main Agent confirms the breakdown spec once with the user and passes it to `profile-analyzer` (`model-infer-perf-breakdown`) to produce both **time distribution** and **per-operator measured-vs-theoretical gap** reports. This spec is reused across all subsequent re-profiling rounds.
5. **Parallel Multi-Source Candidate Discovery (`candidate` subagents)**: Main Agent dispatches one parallel `candidate` subagent per registered source:
   - `multi-stream` $\rightarrow$ `analysis/multi-stream.md` (DAG concurrency decomposition via `model-infer-multi-stream`, proposing $\ge 2$ multi-stream schedules per parallel region).
   - `wiki` $\rightarrow$ `analysis/wiki.md` (queries CANN-Infer-Wiki for model-applicable optimizations).
   - `perf-insight` $\rightarrow$ `analysis/perf-insight.md` (maps contiguous Vector chains $\rightarrow$ fusion, redundant MTE transfers $\rightarrow$ prefetch, top theoretical gap operators $\rightarrow$ kernel tuning).
6. **Dashboard Initialization (`Main Agent`)**: Deduplicates candidate items, resolves **cross-source mutual exclusion groups** vs. **orthogonal stackable plans**, and initializes `plan-dashboard.md`.

### Step 7: Plan Self-Loop (`TR{N}.1` – `TR{N}.5`)
- **Three-State Machine**: Every Plan is strictly in `待实现` (Pending/Needs further tuning), `通过` (Passed accuracy, performance, fallback switch, and side-effect checks), or `淘汰` (Eliminated due to regression, noise-level gain, or replacement by a superior Plan in the same mutual-exclusion group).
- **Global Monotonic `roundN`**: Every attempt on any Plan increments a single global `roundN` counter (`round1`, `round2`, ...), binding profiling directories, `plans/plan-<id>.md` entries, and Dashboard rows without ID collision.
- **Append-Only Derivation (`派生`)**: When an `implementer` or `reviewer` discovers a better variant or narrower cut during `roundN`, the original Plan is never overwritten; a new `plan-<id>` is appended with `待实现` status and explicit lineage (`derived_from: Plan-X / roundN`).
- **TaskList Granularity**: Never create a single permanent "Run Plan Loop" task; create a discrete `TR{N}.1–TR{N}.5` task group per round and close it before opening `TR{N+1}`.

### Step 8: Non-Premature Termination & Final Acceptance (`TF1` – `TF5`)
Only when zero `待实现` Plans and zero viable derivations remain, trigger a mandatory final profiling capture and verify `TF1–TF5` against `round0` under the identical `model-infer-perf-breakdown` spec (never using raw `time.perf_counter` wall-clock numbers, which conflate single-stream busy time with multi-stream overlap).

## Three-File State Spindle (Single Source of Truth per Concern)

| File Path | Concern Owned | Exclusive Writer | Readers |
| :--- | :--- | :--- | :--- |
| `optimization-analysis/<case>/plan-dashboard.md` | **Global State Spine**: Plan status table, mutual-exclusion groups, active adopted set, round index | **Main Agent ONLY** (mirrors subagent summaries immediately after each round) | Main Agent (for scheduling); Subagents (read-only) |
| `{model_dir}/progress.md` | **Shared Context & Process Log**: Scenario baseline, implementation notes, debugging pitfalls | Main Agent (Permanent Zone); Subagents (append to Active Workspace) | All Subagents read first upon entry |
| `optimization-analysis/<case>/plans/plan-<id>.md` | **Per-Plan Technical Spec & Round Ledger**: Domain design, code diff summary, review evidence, derivation proposals | Assigned `candidate`, `implementer`, and `reviewer` for `plan-<id>` | Main Agent reads to adjudicate `通过` / `淘汰` |
