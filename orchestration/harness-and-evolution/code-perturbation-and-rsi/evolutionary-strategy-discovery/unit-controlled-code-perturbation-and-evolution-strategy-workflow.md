---
category: workflow
conditions:
  knowledge_collections:
  - controlled_code_perturbation (180 cards, 10 families)
  - evolution_strategies (P/D/A/R 161 cards + X 89 discoveries)
  repository: cann/cannbot-knowledge
  revision: be5882ef29fa9943305085652d2025a038ba491c
created_at: '2026-10-21'
domain_path: orchestration/harness-and-evolution/code-perturbation-and-rsi/evolutionary-strategy-discovery
estimate_tokens: 620
evidence_strength: software_contract
id: unit-controlled-code-perturbation-and-evolution-strategy-workflow
inherits_from:
- unit-stagnation-triggered-debiased-subcontext-exploration-workflow
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/optimizations/controlled_code_perturbation/index.md#L1-L16
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/optimizations/evolution_strategies/index.md#L1-L9
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/plugins-community/collaborative-agent-kernel-evolution/skills/cake-evo/SKILL.md#L1-L180
  promotion_status: verified
  source_locator: cann/cannbot-knowledge@be5882ef29fa9943305085652d2025a038ba491c:knowledge/contrib/ascendc/optimizations/controlled_code_perturbation/index.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Executes single-variable controlled code perturbation across 10 Ascend C
  operator families and promotes runtime evolutionary discoveries (X-series) into
  curated P/D/A/R optimization cards.
tier: L3
title: Controlled Code Perturbation Across Ten Operator Families and Evolutionary
  Strategy Promotion Workflow
triggers:
  error_signatures:
  - controlled_code_perturbation
  - evolution_strategies
  - cake-evo
  keywords:
  - single-variable code perturbation
  - X-series discovery promotion
  - ten operator families
  - collaborative-agent-kernel-evolution
updated_at: '2026-10-21'
---
# Controlled Code Perturbation Across Ten Operator Families and Evolutionary Strategy Promotion Workflow

## Core Principle

When evolving high-performance Ascend C kernels (`cake-evo` / `collaborative-agent-kernel-evolution`), simultaneous multi-axis rewrites make it impossible to tell which code change caused a speedup or accuracy regression. Reliable kernel evolution combines **Controlled Code Perturbation** (single-variable ablation across 10 canonical operator families) with a **two-tier Evolutionary Strategy Flywheel** that promotes verified runtime discoveries (`X`-series) into canonical `P/D/A/R` strategy cards.

## Stage 1: Classify Target Operator into the Ten Perturbation Families

Before selecting mutation operators, map the target kernel into its primary computational paradigm under `optimizations/controlled_code_perturbation/` (`180` empirical cards):

| Operator Family | Representative Kernels | Dominant Bottleneck & Primary Perturbation Axis |
| --- | --- | --- |
| `elementwise_unary` (54 cards) | `reciprocal`, `gelu`, `rsqrt`, `swish`, `cast`, `cos`, `fill` | Scalar loop overhead, `TQue` buffer depth, vector instruction replacement (`Newton-Raphson` vs `Reciprocal`) |
| `elementwise_binary` (41 cards) | `add`, `sub`, `mul`, `div`, `maximum`, broadcast ops | Broadcast axis coalescing, UB tile size maximizing 32B DMA bursts, in-place buffer reuse |
| `elementwise_foreach` (13 cards) | Multi-tensor batch elementwise | Host metadata packing and multi-tensor pointer table amortization |
| `maskpredicate` (25 cards) | `where`, `masked_fill`, `clip_by_value_v2`, `isfinite` | Vector compare (`vcmp`) bitmask packing vs scalar branch elimination |
| `fusedcomposite` (15 cards) | `mul_sigmoid`, `add_rms_norm`, `rotary_embedding` | Eliminating intermediate GM write/read via UB `VECCALC` chaining |
| `layouttransform` (11 cards) | `transpose`, `permute`, `contiguous`, `gather`, `eye` | VNCHW (`TransDataTo5HD`) hardware transpose vs strided `DataCopyPad` |
| `argreduce` (10 cards) | `argmax`, `argmin`, `reduce_sum`, `reduce_max` | Two-level (`ReduceMax`/`WholeReduceMax` + `BlockReduceMax`) vs scalar tail scan |
| `interpolation` (7 cards) | `upsample_bilinear`, `grid_sampler`, `lin_space` | Vectorized coordinate weight precomputation and gather coalescing |
| `atomicupdate` (2 cards) | `scatter_add`, `index_add` | `SetAtomicAdd` contention reduction via UB local accumulation when indices cluster |
| `contraction` (2 cards) | `matmul`, `bmm`, `einsum` | L1 residency (`IBShare`), L0A/L0B ping-pong, and `Fixpipe` L0C->GM/UB fusion |

## Stage 2: Single-Variable Perturbation & Bottleneck Attribution Loop

In each evolution round (`cake-evo`):
1. **Isolate One Perturbation Axis**: Mutate **at most one architectural dimension per candidate** (`tiling_tile_size`, `buffer_num_pingpong`, `vector_intrinsic_chain`, `core_split_axis`, or `multi_launch_decomposition`).
2. **3-Gate Verification**:
   - **Anti-Reward-Hacking Gate**: Verify zero `torch` compute fallbacks or cached outputs (`SUB-BEH-001..007`).
   - **100% Case Accuracy Gate**: Run `batch_precision_check.py`; any single-case accuracy failure rejects the perturbation immediately.
   - **Geomean / HAP Speedup Gate**: Record per-shape latency delta (`before_us -> after_us`) and PMU pipe shift (`mte2_ratio`, `vec_ratio`, `mac_ratio`, `scalar_ratio`).

## Stage 3: Two-Tier Promotion (`discovered/X*` $\to$ `cards/P*`)

1. **Runtime Discovery Log (`evolution_strategies/discovered/x*.md`)**: During an active evolution campaign, newly discovered multi-shape winning perturbations are logged immediately as `X`-series records with raw before/after benchmark tables.
2. **Canonical Strategy Promotion (`evolution_strategies/cards/p*.md`)**: Once an `X`-series discovery is validated across $\ge 2$ distinct operators or hardware generations (`a2` / `a3`), abstract its reusable C++ skeleton, precondition guards, and grep search signatures into a `P`-series card (`status: stable`) and mark the original `X`-series record `status: deprecated` with `replaced_by: ../cards/p<id>_*.md` (e.g., `X17..X20 -> P128..P131`).
