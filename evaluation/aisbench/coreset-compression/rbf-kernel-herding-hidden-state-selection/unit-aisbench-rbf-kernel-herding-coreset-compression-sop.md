---
schema: mindie-graded-unit/1
id: unit-aisbench-rbf-kernel-herding-coreset-compression-sop
revision: 1
title: "AISBench Evaluation Coreset Compression: Hidden-State Extraction and RBF Kernel Herding SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: evaluation/aisbench/coreset-compression/rbf-kernel-herding-hidden-state-selection
inherits_from:
  - unit-cann-bench-hap-saturation-and-five-level-curriculum-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: evaluation
triggers:
  error_signatures:
    - "herding_coreset_selector"
    - "python -m herding"
    - "vllm_api_general_chat"
  keywords:
    - "aisbench"
    - "coreset"
    - "rbf-kernel-herding"
    - "dataset-compression"
    - "aime2025"
    - "gpqa"
provenance:
  source_type: source_contract
  source_locator: Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/benchmark/aisbench-dataset-compression-herding/SKILL.md
  evidence_refs:
    - https://github.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/skills/benchmark/aisbench-dataset-compression-herding/SKILL.md#L1-L321
  supersedes: []
---

## Falsifiable Claim

Repeatedly evaluating full reasoning benchmarks (`aime2025`, `gpqa`) across multi-round quantization or kernel tuning creates prohibitive multi-hour evaluation bottlenecks, whereas uniform random subsampling distorts difficulty distribution. Running AISBench `tools/herding_coreset_selector` (`python -m herding`) extracts sample-level hidden-state representations using a local reference LLM ($\le 7\text{B}$ parameters) and applies **RBF Kernel Herding** to select a representative coreset (default `--coreset-ratio 0.2`) that preserves full-dataset Embedding distribution while cutting per-round evaluation time by $5\times$.

## Mechanism & Preconditions

1. **Supported Datasets & Schema Invariants**:
   - `--eval-dataset aime2025`: Requires `<dataset-path>/aime2025.jsonl`.
   - `--eval-dataset gpqa`: Requires `<dataset-path>/gpqa_diamond.csv` containing `question`, `A`, `B`, `C`, `D` columns.
2. **Local Reference Feature-Extraction Model (`--model-path`)**:
   - Herding computes hidden-state embeddings over prompt samples using a local HuggingFace Transformers model (recommended $\le 7\text{B}$ parameters, such as `Qwen3-4B-Instruct-2507` or `Qwen2.5-7B-Instruct`, to bound memory footprint).
   - Because tokenizer and hidden-state geometry depend on `--model-path`, the output directory is namespaced by `<model-name>` (`<output-dir>/<eval-dataset>/herding/<model-name>/`).
3. **Runtime Cost Disclosure**:
   - The NumPy RBF Kernel Herding solver runs on CPU and takes approximately 30 minutes for AIME/GPQA; always check whether a pre-generated coreset already exists before launching compression.

## Standard Operating Procedure (SOP)

### Step 1: Verify `aisbench` & `herding_coreset_selector` Presence

Verify that `aisbench` is installed (`pip list | grep aisbench`) and that `<aisbench-code-path>/tools/herding_coreset_selector/herding/` exists.

### Step 2: Run RBF Kernel Herding Coreset Selection

Change directory into `tools/herding_coreset_selector` and invoke `python -m herding` with explicit CLI arguments:

```bash
cd <aisbench-code-path>/tools/herding_coreset_selector

python3 -m herding \
  --eval-dataset gpqa \
  --dataset-path <aisbench-code-path>/ais_bench/datasets/gpqa \
  --model-path <local_model_dir> \
  --coreset-ratio 0.2 \
  --output-dir ./datasets
```

### Step 3: Verify Output Layout & `indices.json` Provenance

Inspect the generated directory tree under `./datasets/<eval-dataset>/herding/<model-name>/`:
- `origin/<dataset_file>` + `origin/indices.json`: Full normalized dataset and original 0-based sample indices.
- `coreset/<dataset_file>` + `coreset/indices.json`: Selected subset matching `--coreset-ratio` (e.g., 20% retained) and exact source indices in `origin` for reproducibility auditing.

### Step 4: Deliver Evaluation Artifacts by Target Mode

- **Mode A (Standalone Fullset vs. Subset Benchmarking)**:
  Generate paired `run_<dataset>_fullset.sh` and `run_<dataset>_subset.sh` scripts targeting `vllm_api_general_chat` with `--num-warmups 0`. Probe the inference server health endpoint (`curl http://127.0.0.1:<port>/health`) before executing `ais_bench`.
- **Mode B (Quantization Orchestrator Integration)**:
  Register a separate coreset dataset config file under `ais_bench/benchmark/configs/datasets/` (never overwriting the original fullset config) and return both `fullset_config_name` and `coreset_config_name` for `evaluation.yaml` switching.
