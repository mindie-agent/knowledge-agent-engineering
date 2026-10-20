---
category: workflow
conditions:
  component: cannbot/harness/workflow-rollback-advice
  repository: cann/cannbot
  revision: d0f8c40ebc208fa17323781c89e4ec6ea16e4c52
  workload:
  - Cross-task DAG rollback forensics, CoW checkpoint restoration, and upstream redo
    guidance generation
created_at: '2026-10-21'
domain_path: orchestration/harness-and-evolution/state-machine-and-rollback/rollback-advice-and-recovery
estimate_tokens: 600
evidence_strength: software_contract
id: unit-harness-rollback-advice-forensics-and-history-recovery-workflow
inherits_from:
- unit-harness-workflow-orchestrator-approval-and-rollback-advice-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot/blob/d0f8c40ebc208fa17323781c89e4ec6ea16e4c52/harness/workflow-rollback-advice/SKILL.md#L1-L120
  - https://gitcode.com/cann/cannbot/blob/d0f8c40ebc208fa17323781c89e4ec6ea16e4c52/harness/workflow-orchestrator/scripts/orchestrator.py#L245-L378
  promotion_status: verified
  source_locator: cann/cannbot@d0f8c40ebc208fa17323781c89e4ec6ea16e4c52:harness/workflow-rollback-advice/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: 'Executes cross-task DAG rollback recovery: draining in-flight tasks, snapshotting
  failed workspace state to .workflow/history/, running read-only forensics for the
  4-section Rollback Advice, and restoring CoW checkpoints.'
tier: L3
title: Harness Workflow Rollback Advice Forensics, CoW Snapshot Recovery, and Four-Section
  Advice Workflow
triggers:
  error_signatures:
  - advice_pending
  - Rollback Advice four sections
  - 'on_exhaust: rollback'
  keywords:
  - copy_tree clear_tree .workflow
  - rollback_to
  - --reflink=auto
  - finish_advice
updated_at: '2026-10-21'
---
# Harness Workflow Rollback Advice Forensics, CoW Snapshot Recovery, and Four-Section Advice Workflow

## Core Principle

When a downstream task $X$ exhausts its retry budget (`retries >= max_retries`) with `on_exhaust: rollback` targeting an upstream design/coding task $Y$ (`rollback_to: Y`), simply resetting $Y$ to `pending` and wiping the workspace causes $Y$ to **repeat the exact same design flaw** that doomed $X$. The orchestrator bridges rollbacks via an automated **Forensic Advice Phase (`phase: advice`)** and **Copy-on-Write (`cp -a --reflink=auto`) workspace checkpointing** that preserves `.workflow/` ledger integrity.

## 4-Stage Cross-Task Rollback & Forensic Advice Workflow

### Stage 1: Checkpoint Archival & `.workflow` Ledger Exclusion (`copy_tree` / `clear_tree`)

1. **Snapshot Exclusion Invariant**: Both checkpoint creation and rollback restoration (`copy_tree` and `clear_tree` in `orchestrator.py`) **must skip `.workflow` (`WORKFLOW_DIR_NAME`)**. Overwriting `.workflow/` during workspace restoration would destroy `status.json`, `log.jsonl`, and the session history of the very failure being recovered!
2. **CoW Zero-Copy Archival**: Before wiping the failed workspace state of $X$, archive the current workspace tree (excluding `.workflow`) into `.workflow/history/<X>.to.<Y>.<n>/` using `cp -a --reflink=auto` and restore the pre-$Y$ checkpoint snapshot.

### Stage 2: Read-Only Forensic Investigation (`workflow-rollback-advice`)

Before re-running upstream task $Y$, the orchestrator dispatches an `advice` task (`advice_pending` in `status.json`). The advice agent operates under **strict read-only discipline on the ledger and source code** (writing **only** the target advice file):
1. **Inspect Failure Verdicts & Logs**: Read `.workflow/log.jsonl` and `.workflow/verdicts/history/<X>.*.json` to extract the exact verifier rejection reasons across all retry attempts of $X$.
2. **Tail Session Transcripts**: Read the tail (`100..200` lines) of `.workflow/sessions/<X>.execute.*.jsonl` and `<X>.verify.*.jsonl` (including appended `exit=<rc>` and `stderr` crash trailers).
3. **Diff Archived Failed Tree**: Inspect `.workflow/history/<X>.to.<Y>.<n>/` to see what upstream task $Y$ produced and how downstream task $X$ failed when consuming it (e.g., $Y$ chose a tiling layout or interface signature that exceeded UB capacity in $X$).

### Stage 3: Mandatory Four-Section `# Rollback Advice: <X> -> <Y>` Deliverable

Write the advice document to the exact path specified by `entry["advice"]` (`.workflow/advice/<X>.to.<Y>.<n>.md`) using this four-section structure (`< 80` lines total):
1. **`## 失败根因` (Root Cause)**: Why $X$ failed across retries and which specific artifact, interface, or assumption produced by upstream task $Y$ caused the failure.
2. **`## 值得保留` (What to Preserve)**: Validated code fragments, tiling formulas, or passing cases inside `.workflow/history/<X>.to.<Y>.<n>/` that worked and can be reused.
3. **`## 应避免` (What to Avoid)**: Dead-end approaches and specific parameters tried during $X$'s failed retries.
4. **`## 重做建议` (Actionable Redo Plan for $Y$)**: Concrete changes $Y$ must make in its next execution pass so $X$ succeeds downstream.

### Stage 4: Advice Consumption & Upstream Re-Dispatch (`finish_advice`)

`finish_advice()` verifies that the advice Markdown file exists and is non-empty (`result="ready"`), after which `get_task.py` injects the advice file path directly into upstream task $Y$'s redo prompt.
