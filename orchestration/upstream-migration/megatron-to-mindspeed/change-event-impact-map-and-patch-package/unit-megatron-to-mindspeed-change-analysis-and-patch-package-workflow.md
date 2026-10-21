---
schema: mindie-graded-unit/1
id: unit-megatron-to-mindspeed-change-analysis-and-patch-package-workflow
revision: 1
title: "Upstream Megatron-LM to MindSpeed Migration: Feature Event Clustering, Impact Mapping, and Patch Package Workflow"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: orchestration/upstream-migration/megatron-to-mindspeed/change-event-impact-map-and-patch-package
inherits_from:
  - unit-harness-workflow-dag-state-machine-and-approval-gate-workflow
  - unit-superpowers-review-request-git-range
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: orchestration
triggers:
  error_signatures:
    - "megatron-change-analyzer"
    - "megatron-impact-mapper"
    - "megatron-migration-generator"
    - "implementation_units"
  keywords:
    - "megatron-to-mindspeed"
    - "change-analyzer"
    - "impact-mapper"
    - "migration-generator"
    - "patch-series"
provenance:
  source_type: source_contract
  source_locator: Ascend/agent-skills@942f0138aa9c78467507babb09aa9c64c5f534dc:skills/megatron-change-analyzer/SKILL.md
  evidence_refs:
    - https://github.com/Ascend/agent-skills/blob/942f0138aa9c78467507babb09aa9c64c5f534dc/skills/megatron-change-analyzer/SKILL.md#L1-L99
    - https://github.com/Ascend/agent-skills/blob/942f0138aa9c78467507babb09aa9c64c5f534dc/skills/megatron-migration-generator/SKILL.md#L1-L89
  supersedes: []
---

## Falsifiable Claim

Porting upstream `Megatron-LM` commits directly into Ascend `MindSpeed` from raw `git log` history produces incomplete single-file edits that miss coupled CLI arguments, wrapper hooks, and lifecycle cleanup. Decomposing upstream evolution into a four-stage pipeline (`megatron-commit-tracker` $\rightarrow$ `megatron-change-analyzer` $\rightarrow$ `megatron-impact-mapper` $\rightarrow$ `megatron-migration-generator`) clusters low-level commits into structured feature events across five implementation-unit surfaces and produces reviewable multi-patch packages (`full.patch`, split `patches/`, and explicit uncovered scope) before touching the local workspace.

## Four-Stage Upstream Adaptation Pipeline

### Stage 1: Branch Alignment & Normalized Change-Set (`megatron-commit-tracker`)
- Establish the exact `(MindSpeed branch, Megatron base branch, Megatron target branch/change-set)` tuple before analyzing diffs. Never guess branch pairings.

### Stage 2: Feature Event Clustering & Five-Surface Decomposition (`megatron-change-analyzer`)
1. Collapse related low-level commits serving a single feature into one high-level `event` while preserving `primary_commit` and `upstream_changed_files` as primary implementation evidence (`build_feature_events.py`).
2. Decompose every migration-relevant event (`migration_relevance: high|medium`) into explicit `implementation_units` across five integration surfaces:
   - `config_surface`: CLI flags, `arguments.py`, and `training_config.py` schema additions/renames.
   - `runtime_logic`: Core distributed execution, parallelism, checkpointing, or forward/backward state changes.
   - `wrappers`: MindSpeed feature patch decorators, monkey-patches, or adapter shims.
   - `tests`: Unit and distributed integration test updates.
   - `lifecycle`: Initialization, memory cleanup, or process-group teardown hooks.
3. Filter out internal upstream refactors that do not alter integration surfaces or are already covered in MindSpeed.

### Stage 3: Downstream Target Mapping (`megatron-impact-mapper`)
- Map each upstream `implementation_unit` to its corresponding MindSpeed adaptation file/hook target, separating high-confidence mappings from speculative items (`impact_report.json`).

### Stage 4: Implementation-First Migration Package Generation (`megatron-migration-generator`)
1. **Default Safe Mode (`report + patch`)**:
   - Do NOT apply direct workspace edits or auto-commits by default.
   - Run `synthesize_full_patch_series.py` and `render_migration_artifacts.py` to generate:
     - `migration_report.md` and `candidate_patch.md` summary index.
     - `features/<feature_slug>/` containing `full.patch`, unit-split `patches/`, `candidate.patch`, `checklist.md`, `package_manifest.json`, and `upstream_reference.json`.
2. **Anti-Placeholder Completeness Contract**:
   - Never represent a multi-surface upstream commit (spanning `config_surface` + `runtime_logic` + `wrappers`) as a single-file placeholder edit.
   - If any `implementation_unit` cannot be synthesized from available context, record it explicitly under uncovered scope and manual follow-ups rather than claiming full feature parity or verified numerical/training accuracy.
