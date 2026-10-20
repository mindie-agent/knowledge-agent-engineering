---
category: workflow
conditions:
  component: cannbot/harness/workflow-orchestrator
  providers:
  - opencode
  - cannbot
  - claude
  - pi
  - codex
  repository: cann/cannbot
  revision: d0f8c40ebc208fa17323781c89e4ec6ea16e4c52
created_at: '2026-10-21'
domain_path: orchestration/harness-and-evolution/state-machine-and-rollback/workflow-dag-and-approval-loop
estimate_tokens: 630
evidence_strength: software_contract
id: unit-harness-workflow-dag-state-machine-and-approval-gate-workflow
inherits_from:
- unit-harness-workflow-orchestrator-approval-and-rollback-advice-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot/blob/d0f8c40ebc208fa17323781c89e4ec6ea16e4c52/harness/workflow-orchestrator/scripts/orchestrator.py#L1-L550
  - https://gitcode.com/cann/cannbot/blob/d0f8c40ebc208fa17323781c89e4ec6ea16e4c52/harness/workflow-orchestrator/SKILL.md#L1-L160
  promotion_status: verified
  source_locator: cann/cannbot@d0f8c40ebc208fa17323781c89e4ec6ea16e4c52:harness/workflow-orchestrator/scripts/orchestrator.py
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: 'Orchestrates multi-agent DAG workflows via orchestrator.py: main-thread
  atomic status.json ownership, FIRST_COMPLETED barrier-free scheduling, file-based
  verify verdicts, and poll_workflow approval gates.'
tier: L3
title: Harness Workflow Orchestrator Event-Driven DAG State Machine, Verdict Files,
  and Approval Loop Workflow
triggers:
  error_signatures:
  - '[ALL TASK FINISHED]'
  - '[STUCK]'
  - verdict_missing
  keywords:
  - orchestrator.lock PID liveness
  - FIRST_COMPLETED
  - consume_verdict_file
  - reset_transient
updated_at: '2026-10-21'
---
# Harness Workflow Orchestrator Event-Driven DAG State Machine, Verdict Files, and Approval Loop Workflow

## Core Principle

Multi-agent DAG orchestration (`cannbot/harness/workflow-orchestrator`) must never rely on natural-language parsing of agent stdout to decide verification verdicts or allow worker threads to concurrently mutate `.workflow/status.json`. The orchestrator enforces **single-thread atomic ledger ownership**, **barrier-free `FIRST_COMPLETED` scheduling**, **file-based verification verdicts (`.workflow/verdicts/`)**, and **in-flight drain guards** before terminal exit or rollback.

## 1. Startup Recovery & Single-Instance Lock (`acquire_lock` + `reset_transient`)

1. **PID Liveness Lock (`.workflow/orchestrator.lock`)**: Before entering the main loop, acquire `.workflow/orchestrator.lock`. If a lock file exists, probe the recorded PID with `os.kill(pid, 0)`; automatically take over stale locks from dead processes and refuse double-launch if the PID is alive.
2. **Crash Transient Reset (`reset_transient`)**: When resuming an existing `.workflow/status.json`, reset in-memory transient states **before** polling tasks:
   - `running -> pending`
   - `verifying -> executed`
   Leaving `running` or `verifying` in `status.json` after an orchestrator crash permanently deadlocks the DAG because `get_task.py` treats them as already in-flight.

## 2. Barrier-Free `FIRST_COMPLETED` Scheduling & Main-Thread Ledger Exclusivity

1. **Single Persistent `ThreadPoolExecutor(max_workers=max_parallel)`**:
   - Worker threads execute `run_agent()` only (streaming agent CLI stdout to `.workflow/sessions/<task_id>.<phase>.<timestamp>.jsonl`) and **never read or write `.workflow/status.json`**.
   - The main thread exclusively harvests completed futures (`harvest_done`), invokes `get_task.py`, pre-claims task states (`pending -> running`, `executed -> verifying`), and writes `status.json` via atomic temp-file + `os.replace(tmp, path)`.
2. **Zero Intra-Batch Barrier (`wait(inflight, return_when=FIRST_COMPLETED)`)**:
   - As soon as **any single task** finishes `execute` or `verify`, the main thread harvests it and immediately calls `get_task.py` to fill free worker slots. Fast downstream tasks start immediately without waiting for slow sibling tasks in the same dispatch wave.

## 3. File-Based Verification Verdicts (`clear_verdict_file` / `consume_verdict_file`)

Never trust natural-language words (`"通过"` / `"pass"`) in verifier stdout:
1. Before dispatching a `verify` phase for `task_id`, `clear_verdict_file` archives any stale `.workflow/verdicts/<task_id>.json` into `.workflow/verdicts/history/<task_id>.<timestamp>.json`.
2. The verifier agent must invoke `verdict_pass.py` or `verdict_fail.py --reason "..."` to write `.workflow/verdicts/<task_id>.json` (`{"verdict": "pass"|"fail", "reason": ...}`).
3. If the verifier exits `0` without writing a valid verdict file, `resolve_verdict_reply` logs `verdict_missing` and returns `""`, keeping the task in `verifying` for re-verification rather than silently passing.

## 4. In-Flight Drain Guard & Approval Polling (`poll_workflow.py`)

- **In-Flight Drain Guard**: When `get_task.py` returns a terminal signal (`[ALL TASK FINISHED]` or `[STUCK]`) or triggers a rollback while `ctx.inflight` is non-empty, **stop dispatching new tasks and wait for all in-flight futures to finish and harvest** before executing the rollback or exiting—preventing orphan agent processes from writing into a rolled-back workspace.
- **Human Approval Loop**: When tasks enter `waiting_human` (`approve` gate) and `inflight` is empty, `orchestrator.py` sleeps `APPROVAL_POLL_SECONDS` (`1s`) without incrementing `empty_rounds` while the host agent runs `poll_workflow.py` and relays user `approve` / `redo` / `fail` decisions via `approve_task.py`.
