---
category: axiom
conditions:
  benchmark_suite: CANN-Bench Season 2 (60 operators, L1-L5)
  repository: cann/cann-bench
  revision: 2d7cd332247f33a66ca67c77d9f09d446a780dfe
  scoring_weights: w_c=0.2, w_f=0.3, w_p=0.5
created_at: '2026-10-21'
domain_path: evaluation/cann-bench/hap-and-curriculum/hap-metric-and-curriculum-axioms
estimate_tokens: 600
evidence_strength: software_contract
id: unit-cann-bench-hap-saturation-and-five-level-curriculum-axioms
inherits_from:
- unit-multi-agent-npu-lease-and-container-isolation-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cann-bench/blob/2d7cd332247f33a66ca67c77d9f09d446a780dfe/docs/spec/benchmark_spec.md#L1-L220
  - https://gitcode.com/cann/cann-bench/blob/2d7cd332247f33a66ca67c77d9f09d446a780dfe/README.md#L1-L160
  promotion_status: verified
  source_locator: cann/cann-bench@2d7cd332247f33a66ca67c77d9f09d446a780dfe:docs/spec/benchmark_spec.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Establishes the mathematical axioms of CANN-Bench's Horizon-Anchored Performance
  (HAP) metric, its bounded saturation over unbounded speedup, invalid-anchor None
  guards, and the L1-L5 60-operator curriculum.
tier: L1
title: CANN-Bench HAP Metric Saturation Properties, Invalid-Anchor Guard, and Five-Level
  Curriculum Axioms
triggers:
  error_signatures:
  - EachOperatorScore all-case veto
  - Horizon-Anchored Performance
  - t_anchor_0 <= t_anchor_100
  keywords:
  - HAP saturation
  - invalid anchor guard
  - five-level operator curriculum
  - harmonic mean speedup flaw
updated_at: '2026-10-21'
---
# CANN-Bench HAP Metric Saturation Properties, Invalid-Anchor Guard, and Five-Level Curriculum Axioms

## Core Invariant

Raw speedup ratios ($S = T_{\text{base}} / T_{\text{cand}}$) are mathematically unsuitable for aggregating multi-operator agent benchmarks: a single trivial operator where PyTorch eager mode has $50\times$ launch overhead can dominate an arithmetic or geometric mean across 60 tasks. `CANN-Bench` resolves this via **Horizon-Anchored Performance (`HAP`)**, mapping candidate execution time $t$ against two physical horizon anchors—a slow baseline anchor $t_0$ (`0`-point horizon, typically PyTorch eager) and a hardware/expert ceiling anchor $t_{100}$ (`100`-point horizon, e.g., tuned `aclnn` or theoretical roofline).

## Axiom 1: HAP Mathematical Formulation & Bounded Saturation

For each test case $i$ with valid anchors $t_{0,i} > t_{100,i} > 0$ and candidate latency $t_i > 0$:

$$\text{HAP}_i(t_i) = \text{clip}\left(100 \cdot \frac{t_{0,i} - t_i}{t_{0,i} - t_{100,i}},\; 0,\; 100\right) \quad (\text{or log-ratio / harmonic-anchored variant per season spec})$$

1. **Zero-Floor Gate**: If $t_i \ge t_{0,i}$ (slower than or equal to the baseline anchor), $\text{HAP}_i = 0$.
2. **Ceiling Saturation**: As $t_i \to t_{100,i}$, $\text{HAP}_i \to 100$, preventing a single outlier speedup on a poorly written baseline from inflating the suite-wide score.
3. **Invalid-Anchor `None` Guard**: If an environment calibration produces degenerate anchors ($t_{0,i} \le t_{100,i}$, $t_{0,i} \le 0$, or $t_{100,i} \le 0$), the scorer **must return `None` (invalid anchor configuration)** and halt aggregation rather than dividing by zero or inverting the score sign!

## Axiom 2: Three-Stage Hierarchical Operator Score (`EachOperatorScore`)

Each operator's composite score (`0..100`) enforces a strict dependency chain (`Compile -> Correctness -> Performance`) with weights $(w_c, w_f, w_p) = (0.2,\; 0.3,\; 0.5)$:

$$\text{EachOperatorScore} = 100 \cdot \left(0.2 \cdot \mathbb{I}_{\text{compile}} + 0.3 \cdot \mathbb{I}_{\text{compile}} \cdot \mathbb{I}_{\text{functional}} + 0.5 \cdot \mathbb{I}_{\text{compile}} \cdot \mathbb{I}_{\text{functional}} \cdot \frac{\overline{\text{HAP}}}{100}\right)$$

- **All-Case Functional Veto**: $\mathbb{I}_{\text{functional}} = 1$ **if and only if 100% of test cases pass precision verification**. Passing 9/10 cases yields $\mathbb{I}_{\text{functional}} = 0$ and forfeits both the 30 correctness points and all 50 performance points (`EachOperatorScore = 20.0`).

## Axiom 3: Five-Level (L1–L5) 60-Operator Complexity Curriculum

`CANN-Bench` Season 2 structures 60 operators across five progressive architectural complexity tiers so agent capability breaks down cleanly by hardware paradigm:
- **`L1` (Basic Vector / Elementwise & Simple Activation)**: Single-pipe `AIV` `MTE2 -> VEC -> MTE3` double buffering and 32B tail alignment.
- **`L2` (Reduction, Normalization & Index/Mask)**: Tree/whole-reduceUB workspace sizing, Welford/two-pass statistics (`LayerNorm`, `RMSNorm`, `Softmax`), and `DataCopyPad`.
- **`L3` (Cube MatMul & Quantized / Grouped GEMM)**: `L1 -> L0A/L0B -> L0C -> Fixpipe` tiling, `Bmm`, `GroupedMatmul`, and INT8/FP16 mixed precision.
- **`L4` (CV Mixed Fused Attention & Vision Blocks)**: `MIX_AIC` + `MIX_AIV` `CrossCoreSetFlag`/`WaitFlag` pipelining (`FlashAttention`, `SparseFlashAttention`, `TripletAttention`).
- **`L5` (Multi-Launch Complex Pipelines & Distributed/Custom Mechanisms)**: Multi-phase workspace orchestration, dynamic shape tiling, and cross-core state synchronization.
