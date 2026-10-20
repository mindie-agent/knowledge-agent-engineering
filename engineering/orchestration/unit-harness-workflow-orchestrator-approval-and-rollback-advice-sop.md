---
category: sop
conditions:
  repository: cann/cannbot
  revision: d0f8c40ebc208fa17323781c89e4ec6ea16e4c52
domain_path: engineering/orchestration
evidence_strength: software_contract
id: unit-harness-workflow-orchestrator-approval-and-rollback-advice-sop
inherits_from:
- unit-multi-agent-npu-lease-and-container-isolation-axioms
- unit-superpowers-claim-evidence-scope
provenance:
  evidence_refs:
  - docs/evidence/cannbot-sentry-bench-and-harness-provenance.json
  - https://gitcode.com/cann/cannbot/blob/d0f8c40ebc208fa17323781c89e4ec6ea16e4c52/harness/workflow-orchestrator/SKILL.md#L7-L84
  - https://gitcode.com/cann/cannbot/blob/d0f8c40ebc208fa17323781c89e4ec6ea16e4c52/harness/workflow-rollback-advice/SKILL.md#L6-L47
  source_digest: 980f6c3981409af9e5cf8036e29012d9ef0105f99745773a2ea50027e13b8d73
  source_locator: docs/evidence/cannbot-sentry-bench-and-harness-provenance.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: Harness multi-stage workflow orchestration, approval gating, and read-only rollback advice SOP
triggers:
  error_signatures:
  - Unquoted workflow prompt with spaces or parentheses fails inside tmux new-session
  - Protocol-compatible provider selected instead of actual running agent CLI
  - Downstream workflow task exhausts retries and triggers rollback to upstream task
  keywords:
  - workflow-orchestrator
  - workflow-rollback-advice
  - orchestrator.py
  - link_agent_config.py
  - poll_workflow.py
  - approve_task.py
  - Rollback Advice
---
At the pinned `cann/cannbot` revision, `harness/workflow-orchestrator` and `harness/workflow-rollback-advice` govern multi-stage agent workflow execution, human-in-the-loop task approval, and cross-node rollback root-cause recovery (`harness/workflow-orchestrator/SKILL.md:7-84`, `harness/workflow-rollback-advice/SKILL.md:6-47`).

## 1. Workflow Orchestrator Preflight, Detached `tmux` Launch, and Approval Loop (`workflow-orchestrator/SKILL.md:12-84`)

1. **Four-Input Verification & Exact Provider Matching (`Steps 1–3`)**:
   - Query `python3 "$SKILL_ROOT/scripts/orchestrator.py" --help` to obtain the authoritative list of supported `--provider` values.
   - Match `provider` to the **actual CLI binary running the current session** (e.g., `cannbot` when launched via CANNBot, even though its wire protocol is compatible with `opencode`), never inferring the running CLI merely from installed binaries or dotfiles on disk.
   - Canonicalize `workflow.yaml`, `work_dir`, and `$START_DIR` with `realpath -m` (allowing `work_dir` to be created later) and explicitly confirm all four concrete values (`workflow.yaml`, `work_dir`, `provider`, `prompt`) with their full paths/text visible before execution.
2. **Config Linking & Shell-Safe Detached `tmux` Launch (`Steps 4–5`)**:
   - Run `python3 "$SKILL_ROOT/scripts/link_agent_config.py" --start-dir "<start_dir>" --work-dir "<work_dir>" >> "<work_dir>/orchestrator.log" 2>&1` and require exit code `0`.
   - Derive a collision-free tmux session name `orchestrator-<slug>` (appending `-2`, `-3` if `tmux has-session -t <session>` succeeds) and launch `orchestrator.py` inside `tmux new-session -d -s <session>` with double-quoted, shell-escaped (`"`, `` ` ``, `$`) arguments so prompts containing spaces or parentheses cannot fail before log redirection.
   - Poll every ~10s while the tmux session is alive using `python3 "$SKILL_ROOT/scripts/poll_workflow.py" --work-dir "<work_dir>"`, relaying stdout verbatim without wrapper headers or fences, and gating any `Pending Approval` task via `approve_task.py --decision <approve|redo|fail> [--comment "<comment>"]`. Treat `orchestrator.py` strictly as a blackbox judged by its output and exit status.

## 2. Read-Only Cross-Task Rollback Advice Protocol (`workflow-rollback-advice/SKILL.md:6-47`)

When downstream task $X$ exhausts its retry budget and the workflow rolls back to an upstream dependency task $Y$:

1. **Chronological Session & Snapshot Evidence Audit**:
   - Read `.workflow/sessions/<task_id>.<phase>.<timestamp>.jsonl` in task-and-timestamp order across $Y$'s prior `execute` phase, intermediate nodes on the $Y \to X$ dependency chain, and $X$'s failing `execute`/`verify` phases. Inspect non-JSON crash trailers (`exit=` / `stderr` appended at EOF) when a provider CLI terminated abnormally.
   - Cross-check claims against the physical file snapshots archived in `.workflow/history/<X>.to.<Y>.<n>/`.
2. **Four-Section `# Rollback Advice: <X> -> <Y>` Contract**:
   - Write a standalone markdown report containing: (1) **Failure Root Cause** (direct failure + upstream root cause in $Y$ with exact file, line, and log citations), (2) **What to Preserve** (validated artifacts and patterns from the prior generation), (3) **What to Avoid** (invalid assumptions or code paths that caused $X$ to fail), and (4) **Prioritized Redo Plan for $Y$** (concrete file and command instructions so $Y$'s re-execution passes $X$ on the first attempt).
   - **Read-Only Ledger Invariant**: Never mutate any file inside `work_dir` or `.workflow/` (sessions and history archives are immutable audit records) other than writing the target advice markdown path.
