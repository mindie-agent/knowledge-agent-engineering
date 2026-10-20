---
category: sop
conditions:
  device_visibility_variable: ASCEND_RT_VISIBLE_DEVICES
  repository: cann/cann-bench
  revision: 2d7cd332247f33a66ca67c77d9f09d446a780dfe
domain_path: engineering/benchmarking
evidence_strength: software_contract
id: unit-cann-bench-hap-scoring-batch-compile-and-taskunit-retry-sop
inherits_from:
- unit-multi-agent-npu-lease-and-container-isolation-axioms
provenance:
  evidence_refs:
  - docs/evidence/cannbot-sentry-bench-and-harness-provenance.json
  - https://gitcode.com/cann/cann-bench/blob/2d7cd332247f33a66ca67c77d9f09d446a780dfe/README.md#L13-L67
  - https://gitcode.com/cann/cann-bench/blob/2d7cd332247f33a66ca67c77d9f09d446a780dfe/retry_design.md#L1-L249
  source_digest: 1efa2b3a9ad0e5f72d7b79f4e3cc7e10c49c9982673a1b68c0f44a6e071e4a8d
  source_locator: docs/evidence/cannbot-sentry-bench-and-harness-provenance.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: CANN-Bench Hardware-Anchored Performance (HAP) scoring, batch compile gate, and TaskUnit physical-device retry isolation SOP
triggers:
  error_signatures:
  - Child process ASCEND_RT_VISIBLE_DEVICES=0 maps to physical card 0 instead of parent restricted device
  - Single operator build.sh failure zeroes entire multi-operator submission batch
  - Repeatedly hanging test case exhausts TaskUnit retry budget across NPU cards
  keywords:
  - cann-bench
  - HAP
  - Hardware-Anchored Performance
  - EachOperatorScore
  - TaskUnit
  - _physical_device_token_for_task
  - exclude_repeatedly_failed_cases
  - ASCEND_RT_VISIBLE_DEVICES
---
At the pinned `cann/cann-bench` revision, operator generation and optimization benchmarks enforce a 5-level complexity curriculum (`L1` Element-wise, `L2` Norm/Reduction/Gather, `L3` Conv/Pool/MoE, `L4` Attention/RNN, `L5` Stateful/Sparse Attention & Tensor Programs such as `KimiDeltaAttention`, `LactTttChunk`, `TensorProgramVm`), a Hardware-Anchored Performance (`HAP`) metric, a strict batch compilation gate, and `TaskUnit` fault-isolated retry scheduling (`README.md:13-67`, `retry_design.md:1-249`).

## 1. Hardware-Anchored Performance (`HAP`) & Composite Operator Scoring (`README.md:36-67`)

1. **Per-Case `HAP` Formula**:
   Rather than using unbounded relative speedup over a slow composite baseline, `cann-bench` anchors each case $i$ against its hardware theoretical latency floor $T_{\text{HW},i}$ (`t_hw_us`) and CANN baseline latency $T_{\text{baseline},i}$ (`baseline_perf_us`) loaded per SoC from `<bench>/metadata/<hardware>.json`:
   $$\text{HAP}_i = \frac{T_{\text{baseline},i} - T_{\text{HW},i}}{(T_{\text{cand},i} - T_{\text{HW},i}) + (T_{\text{baseline},i} - T_{\text{HW},i})}$$
   - **Regime Semantics**: $T_{\text{cand},i} > T_{\text{baseline},i} \implies \text{HAP}_i \in (0, 0.5)$; $T_{\text{cand},i} = T_{\text{baseline},i} \implies \text{HAP}_i = 0.5$; $T_{\text{cand},i} = T_{\text{HW},i} \implies \text{HAP}_i = 1.0$; $T_{\text{cand},i} < T_{\text{HW},i} \implies \text{HAP}_i > 1.0$ (unclipped).
   - **Invalid-Anchor Guard**: Whenever $T_{\text{cand},i} \le 0$, $T_{\text{HW},i} \le 0$, or the denominator $(T_{\text{cand},i} - T_{\text{HW},i}) + (T_{\text{baseline},i} - T_{\text{HW},i}) \le 0$, `HAP` returns `None` (excluding the case from the performance sum rather than emitting negative or $\pm\infty$ scores).
2. **Composite Operator Score (`w_c = 0.2, w_f = 0.3, w_p = 0.5`)**:
   $$\text{EachOperatorScore} = \left[ 0.2 \cdot \delta_{\text{pass}} + \sum_{i \in \text{cases}} \frac{\delta_{\text{accuracy},i} (0.3 + 0.5 \cdot \text{HAP}_i)}{|\text{cases}|} \right] \cdot 100$$
3. **Batch Compilation Failure Zeroing Gate (`README.md:63-64`)**:
   - When a multi-operator submission is compiled via `build.sh`, **any single operator compile failure causes all operators in the submission batch to score `0`** (`_compile.log` is recorded and the evaluator exits non-zero). The harness intentionally refuses to move or iteratively re-compile subsets of user source files so evaluation remains side-effect-free and embarrassingly parallel. Always verify `bash build.sh` locally before submission.

## 2. `TaskUnit` Partial Recovery, Bad-Case Quarantine, and Physical Device Mapping (`retry_design.md:1-249`)

1. **Partial Recovery & Bad-Case Quarantine (`src/kernel_eval/eval/process_pool.py`)**:
   - When a `TaskUnit` times out, is OOM-killed, or suffers a subprocess crash (`retry_on_timeout=True`, `retry_on_oom=True`, `retry_on_failure=True`), the scheduler recovers already completed cases via `_try_recover_partial_results` and increments `case_failure_count[case_id]` for remaining failed cases.
   - **Threshold-2 Quarantine**: Any case with `case_failure_count[case_id] >= 2` (`exclude_repeatedly_failed_cases=True`) is permanently quarantined as failed so a deterministic hanging kernel cannot poison subsequent retry batches. Remaining cases with `< 2` failures are re-queued onto a different healthy NPU card (`_select_device_for_retry(original_device, excluded_devices, healthy_cards)`).
2. **Logical-to-Physical `ASCEND_RT_VISIBLE_DEVICES` Mapping (`retry_design.md:36-65, 117-134`)**:
   - `task.device_id` is a 0-based **logical** index into the parent process's visible device pool. Setting `env["ASCEND_RT_VISIBLE_DEVICES"] = str(task.device_id)` directly in a child worker is a critical bug when the parent runs under restricted visibility (e.g., parent `ASCEND_RT_VISIBLE_DEVICES=6,7` with logical `task.device_id=0` would bind the child to physical NPU `0` instead of physical NPU `6`).
   - Always resolve the physical token via `_visible_device_tokens_from_env(env)` and map `physical_device = visible_tokens[task.device_id]` before spawning the child process.
