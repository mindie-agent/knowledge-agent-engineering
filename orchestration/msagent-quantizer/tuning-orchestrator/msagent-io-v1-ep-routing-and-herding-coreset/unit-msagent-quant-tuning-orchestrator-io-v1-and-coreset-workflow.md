---
schema: mindie-graded-unit/1
id: unit-msagent-quant-tuning-orchestrator-io-v1-and-coreset-workflow
revision: 1
title: "msagent Automated Quantization Orchestrator: MSAGENT_IO v1 Protocol, Multi-Card EP Routing, and Coreset Tuning Loop"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: orchestration/msagent-quantizer/tuning-orchestrator/msagent-io-v1-ep-routing-and-herding-coreset
inherits_from:
  - unit-harness-workflow-dag-state-machine-and-approval-gate-workflow
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: orchestration
triggers:
  error_signatures:
    - "msagent.subagent_io"
    - "msagent-io v1"
    - "VBENCH_ARGS_INVALID"
    - "[EP_CHECK]"
  keywords:
    - "quantization-accuracy-tuning-orchestrator"
    - "msagent-io-v1"
    - "ep-parallel-adaptation"
    - "apply-rollback"
    - "dit-quantization"
provenance:
  source_type: source_contract
  source_locator: Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/quantizer/quantization-accuracy-tuning-orchestrator/SKILL.md
  evidence_refs:
    - https://github.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/skills/quantizer/quantization-accuracy-tuning-orchestrator/SKILL.md#L1-L255
    - https://github.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/skills/quantizer/quantization-accuracy-tuning-orchestrator/references/subagent_io_protocol.md#L1-L199
  supersedes: []
---

## Falsifiable Claim

In `msagent` automated quantization and accuracy tuning (`quantization-accuracy-tuning-orchestrator`), unstructured natural-language subagent handoffs, unadapted multi-card MoE quantization, or unbounded checkpoint writes cause silent evaluation mismatches and disk exhaustion. Enforcing the machine-readable `msagent-io v1` envelope contract across all nine subagents, routing $\ge 2$ NPU MoE jobs through `msmodelslim-ep-parallel-adaptation` (`[EP_CHECK]`), executing a "subset tune $\rightarrow$ fullset verify $\rightarrow$ fullset fallback" accuracy loop, and capping on-disk quantized checkpoints to at most 2 copies guarantees deterministic convergence across `llm`, `vlm_text`, and `dit` model families.

## 1. `MSAGENT_IO v1` Subagent Envelope Contract

Every delegation from the orchestrator (`task.description`) and every final reply from a subagent MUST contain at most $\le 3$ lines of plain-text summary outside the fence and **exactly one** ` ```msagent-io v1 ` JSON block:

1. **Delegation Envelope (`Main Agent -> Subagent`)**:
   - Contains `"protocol": "msagent.subagent_io"`, `"subagent_type"`, and `"input": { ... }`.
   - Must NOT contain `status`, `output`, or `error`. Outside-fence text must never paste full YAML files or repeat `input` fields.
2. **Response Envelope (`Subagent -> Main Agent`)**:
   - Contains `"protocol": "msagent.subagent_io"`, `"subagent_type"`, and `"status": "ok" | "failed"`.
   - When `"status": "ok"`, populates `"output": { ..., "commands": [{"name": "...", "command": "..."}] }` and omits `"error"`.
   - When `"status": "failed"`, populates `"error": {"code": "...", "message": "..."}` and omits `"output"`.
   - **Hard Gate**: If a subagent reply lacks a valid `msagent-io v1` block, the orchestrator must treat the step as failed or retry—never infer success from prose tables.

## 2. Routing by Hardware Card Count & `model_family`

### Multi-Card ($\ge 2$ NPUs) MoE EP Parallel Adaptation
When the user supplies $\ge 2$ NPU device IDs (e.g., `npu:0,1,2,3`):
- Between **Model Preparation** and **Quantization Tuning**, the orchestrator delegates to `msmodelslim-ep-parallel-adaptation` to check whether the model is MoE and verify EP readiness (`[EP_CHECK]`).
- If `requires_ep == true`, every subsequent quantization round and evaluation service MUST run in multi-card EP mode and contain `[EP_CHECK]` in its quantization log—never falling back to single-card or DP mid-loop.

### `model_family` Dispatch Matrix (`analysis_result.yaml`)
- **`llm` / `vlm_text`**: Routes through `quant-tuning-analyze-{llm,vlm}` $\rightarrow$ `quant-tuning-quantize-{llm,vlm}` (`apiversion: modelslim_v1`) $\rightarrow$ `quant-tuning-evaluate`. Uses `quantization-expert-experience-tuning-rules` prior to binary layer-rollback search under `standing_high_with_experience`.
- **`dit` (Diffusion Transformer)**: Routes through `msmodelslim-model-analysis` $\rightarrow$ `msmodelslim-model-adapt` $\rightarrow$ `quant-tuning-quantize-dit` (`apiversion: multimodal_sd_modelslim_v1`).
  - Layer rollback invokes `quantization-expert-experience-tuning-rules/scripts/apply_rollback.py` (`--first-n-blocks`) directly via `execute` without spawning a subagent.
  - Video generation runs via `quant-tuning-evaluate` (DiT extension with 60s liveness heartbeats and no fixed timeout kill; `--ckpt_dir` always points to FP weights while quantized weights pass via `--quant_dit_path`), followed by `quant-tuning-score-dit` (AISBench-VBench), exiting as soon as `is_satisfied == true` (`loss_vs_baseline >= -tolerance`, default `0.05`).

## 3. Coreset Iteration & Two-Checkpoint Disk Governance

1. **Subset $\rightarrow$ Fullset Tuning Loop**:
   - Confirm compressed dataset source (user-supplied coreset, `aisbench-dataset-compression-herding` RBF Kernel Herding coreset, or fullset fallback) and establish both subset and fullset accuracy exit thresholds before entering the loop.
   - Tune on the compressed coreset until the subset threshold is met, then run a single fullset verification; if fullset verification fails, switch directly to fullset tuning (never iteratively tightening subset tolerances by fixed steps).
2. **Strict Two-Checkpoint Storage Cap**:
   - Store at most **2 quantized weight directories** on disk simultaneously: (1) the current iteration checkpoint and (2) the best passing iteration checkpoint so far.
   - Delete superseded checkpoints immediately using `rm -r` (`rm -rf` is strictly prohibited by orchestrator safety rules).
