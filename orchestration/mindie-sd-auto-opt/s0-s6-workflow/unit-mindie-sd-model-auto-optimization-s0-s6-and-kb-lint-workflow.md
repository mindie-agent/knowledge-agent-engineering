---
schema: mindie-graded-unit/1
id: unit-mindie-sd-model-auto-optimization-s0-s6-and-kb-lint-workflow
title: MindIE-SD model-auto-optimization S0-S6 stage-gated orchestration, multi-agent fan-out, dual-report lint, and kb_lint workflow
tier: L3
category: workflow
evidence_strength: software_contract
domain_path: orchestration/mindie-sd-auto-opt/s0-s6-workflow
inherits_from:
  - unit-multi-agent-npu-lease-and-container-isolation-axioms
  - unit-okf-v02-bundle-profiles-provenance-and-trust-tier-axioms
conditions:
  repository: Ascend/MindIE-SD
  revision: 9fe772dfb90696e21b5e5d4e92ca07dffab84cf4
triggers:
  error_signatures:
    - "stage_gate.py --stage close fails due to missing overview_report.md or detail_report.md"
    - "report_lint.py rejects un-anchored estimates on baseline/recommended/triad rows"
    - "kb_lint.py rejects duplicated thresholds or un-scoped raw wall-clock numbers in skill prose"
  keywords:
    - model-auto-optimization
    - stage_gate
    - report_lint
    - audit_report
    - kb_lint
    - post-enable-review
    - seam_check
    - run-state
provenance:
  source_type: ecosystem_repo
  source_locator: Ascend/MindIE-SD@9fe772dfb90696e21b5e5d4e92ca07dffab84cf4:.agents/skills/model-auto-optimization/SKILL.md
  source_digest: 6a6619b9bd6692cddcd65cce94ca48bd254fbc53eb82ff5f6e938dfdcea17813
  evidence_refs:
    - Ascend/MindIE-SD@9fe772dfb90696e21b5e5d4e92ca07dffab84cf4:.agents/skills/model-auto-optimization/SKILL.md
    - Ascend/MindIE-SD@9fe772dfb90696e21b5e5d4e92ca07dffab84cf4:.agents/README.md
  supersedes: []
revision: 1
---
## Multi-Stage Auto-Optimization & Knowledge Governance Workflow

### Step 1: Initialize `run-state.md` & Upfront Gap Strategy (`§0`)
1. Classify the task into the 3-tier skill hierarchy:
   - **L1 Orchestration**: `model-auto-optimization` (end-to-end model optimization across `S0–S6 + Close`) vs `dev-workflow` (in-repo operator/pattern/test code changes).
   - **L2 Domain Entry**: `performance-optimization` (single source of truth for the $3\%$ noise floor and stop conditions).
   - **L3 Capability Modules**: `env-install`, `dummy-run`, `profiling-collect`, `profiling-analyze`, `fusion-scope-analyze`, `dit-perf-opt`, `dit-parallel-opt`, `vae-opt`, `host-opt`, `pattern-dev`, `operator-dev`, `quantization-dev`, `framework-integration`, `accuracy-gate`, `perf-gate`.
2. Before entering `S0`, scan `framework-support-matrix.md` (`已支持` / `待配置` / `待开发`), confirm the feature coverage checklist and gap-filling policy with the user once, and initialize `{workdir}/agentic/run-state.md`.

### Step 2: Execute Stage-Gated Optimization (`S0 -> S1 -> S3 -> S4 -> S5 -> S6 -> Close`)
At the end of every stage, update `run-state.md` with artifact pointers and run `python scripts/stage_gate.py --stage {Sn} --run-dir {workdir}/agentic` (`error == 0` required before advancing):
1. **`S0` (Environment, Weights & `dummy-run` Baseline)**: Verify `import mindiesd`, weight completeness (no `.incomplete` shards), and fast 1-layer/few-step `dummy-run` execution.
2. **`S1` (DiT Kernel Fusion)**: Run `profiling-collect` $\to$ `profiling-analyze` $\to$ `fusion-scope-analyze` (`check_fusion_scope.py`), then integrate via `framework-integration` (Stage 1 API injection $\to$ Stage 2 `MindieSDBackend` compile) with 3-layer exit evidence.
3. **`S3` (DiT Multi-Card Parallelism & Comm Masking)**: Route to `dit-parallel-opt` to select same-island `UB` card groups, evaluate `USP` / `CFG` / `AllGather-KV x Ulysses` via the DiT-only few-step protocol, and apply comm-stream masking.
4. **`S4` (DiT Lossy Acceleration & 6-Surface `post-enable-review`)**:
   - Evaluate single-point `Cache`, `Sparse` (`rf_v2`/`rf_v3`/`EagleQBSA`), and `Quantization` (`w8a8`/`mxfp8`/`w4a4`), then execute the `[MUST]` pairwise + triad (`Cache + Quant(w8a8f8) + Sparse`) combination search (`seam_check.py`).
   - Whenever a feature alters the kernel sequence, precision domain, or HBM footprint (especially after quantization), run the **6-surface `post-enable-review`** (re-auditing `S1` fusion opportunities such as `mm_swiglu_mxquant` and `S3` parallel/comm ratios on the post-quantization timeline).
5. **`S5` (Train-Aware Few-Step Distillation & Static LoRA Offline Merge)**: Evaluate 4/8-step distillation adapters (verifying `base_schedule` denoiser evaluation count and worker load counters) and merge static serving-time LoRAs offline into base weights when `|Δwall| >= 2 * |Δdev|`.
6. **`S6` (Non-DiT `vae-opt` + `host-opt`)**: Triggered **only** when stage accounting shows `非DiT-解码段` or `非DiT-host段` $\ge 10\%$ of E2EL (otherwise recorded as `skipped` with measured share).

### Step 3: Enforce Multi-Agent Role Isolation & Resource Leases
1. Default to single-agent execution unless the fan-out criteria in `agent-roles-and-handoff.md` are met.
2. When fanning out across parallel `(feature x card_group)` cells:
   - Bind each worker to an isolated same-island NPU card group lease and dedicated port.
   - Enforce **4-role separation**: `coordinator` (owns `run-state.md` and `overview_report.md`), `implementer`, `analyst`, and an **independent `reviewer` instance** that did not participate in implementing the feature (required for `perf-gate` Level-6 evidence and `accuracy-gate` sign-off).

### Step 4: Validate Dual Reports & 13-Rule Knowledge-Base Lint (`report_lint.py` & `kb_lint.py`)
1. **Dual-Report Closure**: Produce both `overview_report.md` (8-column contract with mandatory measured `e2e`, `步数`, and `首步耗时` on baseline, recommended, and triad rows, plus compact 3-part descriptions $\le 350$ chars) and `detail_report.md`. Validate with `report_lint.py` (`error == 0`), `audit_report.py` (`error == 0`), and `stage_gate.py --stage close`.
2. **13-Rule Knowledge Hygiene (`kb_lint.py`)**: Enforce single-point ownership of thresholds/commands, ban raw environment-specific wall-clock numbers in reusable skill prose (archive raw numbers under `{run_results_dir}/archive/` and keep only ratios/criteria in skills), and tag unmerged third-party fork workarounds as `[探针]`.
