---
category: sop
conditions:
  execution_surface: remote_npu_container
  job_execution: background_with_timeout_budget
  source_control: git_worktree
  terminal_effect_classes:
  - completed
  - failed
  - unknown
  verification_scope: affected_scope
domain_path: engineering/remote-execution
evidence_strength: reproduced_empirical
id: unit-remote-npu-container-patch-and-long-job-verification-sop
inherits_from:
- unit-multi-agent-npu-lease-and-container-isolation-axioms
- unit-superpowers-claim-evidence-scope
provenance:
  evidence_refs:
  - docs/evidence/agent-engineering-npu-provenance.json
  - mindie-003-execution@9204f6be8c973adc88c0238456ec60872cc18c072866c85d6c4ad9bbbb797dbd:agy-dsh-waves/wave7/inputs/offline_transcript_digest.md
  - mindie-bench@c42493d6ff916debf2967b76adace62113b41340:AGENTS.md
  - mindie-agent-codex@24049537746b23f9a5fc91412319a73488857a8e:AGENTS.md
  - mindie-agent-codex@24049537746b23f9a5fc91412319a73488857a8e:plugins/mindie-agent/skills/mindie-agent/references/domain-skills.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/collaborative-agent-kernel-evolution/skills/remote-cann-development/SKILL.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/collaborative-agent-kernel-evolution/skills/git-version-management/SKILL.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:infra/gitcode-pr-handler/SKILL.md
  source_digest: d36d173738cc47863d74fddd3772ae381c78503f1884822f02d16add39adda1b
  source_locator: docs/evidence/agent-engineering-npu-provenance.json
  source_type: offline_transcript_and_software_contract
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: Patch and verify remote NPU containers in affected scope with bounded jobs
triggers:
  error_signatures:
  - timeout
  - UnknownExternalEffects
  - in_flight_runs
  - exit code 137
  - SIGKILL
  - rsync failed
  - git apply failed
  - health check failed
  - SSH connection lost
  keywords:
  - remote NPU
  - container
  - git worktree
  - minimal diff
  - fast reproducer
  - single layer
  - long build
  - background job
  - timeout budget
  - cleanup grace
  - completed
  - failed
  - unknown
  - external effects
content_digest: sha256:afad97b5333c740328ed1afe87c5ff6e9401a04ff4411668d0f6ae787e615ac8
---
# Procedure

Use this five-step loop when a patch must be validated against a remote Ascend NPU container and a full model startup is too expensive to use as the first feedback signal. The loop is deliberately scoped: it verifies the affected contract first, then expands only when the evidence justifies the cost.

## Step 1 — Edit and verify only the affected scope in an isolated git worktree

1. Create a dedicated worktree from a recorded base commit. Do not edit the parent checkout or another agent's worktree.
2. Declare the affected scope before editing: the operator, layer, backend, or configuration path under test. Change only files in that scope unless a shared contract forces a wider change.
3. Run the smallest checks that exercise the changed contract: import, compile, symbol registration, unit tests, one operator, one layer, or one graph-capture shape.
4. Record `base_commit`, `git_commit`, `diff_digest`, the changed-file list, the commands, their exit codes, and the exact test scope. A claim about a fix must name the command that exercised the fix.
5. Do not run an unrelated full suite by default. Expand the scope when the change crosses a shared boundary, when a caller contract changed, or when the existing unit tests cannot distinguish the candidate from the baseline.
6. Use `git -C <worktree>` with a realpath guard for every `add` and `commit`. Never use a parent-repository `git add -A` to capture worktree files.

The output of this step is a reviewable diff plus a reproducible affected-scope check. It is not yet evidence that the full model works.

## Step 2 — Sync the minimal diff to the target container

1. Produce a unified diff or an explicit changed-file manifest with SHA-256 hashes. Keep generated artifacts, logs, profiling output, and local caches out of the diff.
2. Apply into a clean staging path in the target container. Run `git apply --check` before `git apply`; if the target has local changes, preserve them explicitly and record the preservation method.
3. Record the before and after hashes of every changed file. If the target repository has a dirty state that the patch does not own, do not overwrite it silently.
4. Do not rsync a whole dirty repository over a live container. Sync only the declared paths, or sync a prepared tree into a run-specific staging directory and switch to it deliberately.
5. Re-check the target commit and environment after the sync. A patch applied to the wrong commit, the wrong container, or the wrong CANN environment is not the same experiment.
6. For multi-file native builds, record the build manifest and the file count/hashes. The recorded live session used this discipline to align 1,960 native files across 12 targets and to exclude rejected work from the distributed baseline.

The output of this step is a target container whose changed bytes are known, not a container that “looks updated.”

## Step 3 — Run fast single-layer or unit reproducers before full model startup

Run the cheapest reproducer that can falsify the change:

1. **Import and registration check.** The module imports, the custom operator registers, the expected symbol is selected, and the fallback path is not silently taken.
2. **Operator or single-layer check.** Use a small shape with the real dtype/layout, including a non-zero offset or a page-boundary case when the change touches cache addressing.
3. **Metadata and graph check.** Exercise the metadata update, cache binding, and one graph capture/replay shape before loading the full model.
4. **Small-model precursor.** When available, use a reduced-expert or reduced-layer model as the first end-to-end signal. In the recorded session this was a 16-expert, 93-layer model used before the main multi-machine model.
5. **Full model only after the above pass.** A service process that merely starts is not a passing service: require a health check, a first real request, and the relevant graph replay or accuracy check.

Use the leased visible-device list. Do not assume device 0, and do not reuse a device whose previous process has not been drained. A fast reproducer that fails is a reason to stop and inspect, not a reason to skip to the full model.

## Step 4 — Manage long builds and benchmarks as background jobs with explicit timeout budgets

Long kernel builds, model loading, graph capture, profiling, and benchmark sweeps must be registered jobs, not unattended shell commands.

Before launching:

- Define the inner budgets: sync, configure, compile, link, package, test, probe, profile, and cleanup.
- Compute the outer budget as `outer_timeout >= sum(inner_budgets) + cleanup_grace`. The inequality is a hard contract, not a suggestion.
- Reserve cleanup time for `SIGTERM` -> bounded drain -> `SIGKILL`, stdout/stderr flush, receipt write, and process/device inventory. A clean test timeout is not `unknown` if the inventory proves there are no residual processes or writes.
- Register `run_id` or `job_id`, slot, endpoint, and manifest path in `PROGRESS.md` under `in_flight_runs` before waiting or ending the turn. A submitted `run_id` is never replaced.
- Poll with a bounded interval or a job-status query. Do not busy-poll, and do not submit a duplicate command because the first one is slow.

A worked budget:

```text
compile            1800 s
package             300 s
single-layer test   600 s
probe               120 s
cleanup              30 s
--------------------------------
sum(inner)         2850 s
outer_timeout      >= 2850 s
```

At resume or after context compaction, query the original `run_id`/`job_id` first. Only remove it from `in_flight_runs` after a terminal receipt is observed and recorded. Long builds may finish out of order; report the exact completed target count rather than a rounded “almost done.”

## Step 5 — Classify the external effect as `completed`, `failed`, or `unknown`

Exit status describes the local command. It does not, by itself, describe the external effect.

| Class | Meaning | Required evidence |
|---|---|---|
| `completed` | The command reached a terminal success state and the expected artifacts/requests exist. | Terminal receipt, exit status, artifact hashes or request output, post-run inventory. |
| `failed` | The command reached a known terminal failure and the post-state is clean. | Failure receipt, non-zero exit or explicit failure verdict, inventory proving no residual process or unauthorized write. |
| `unknown` | The post-state is indeterminate: timeout without inventory, SSH/container loss, local receipt failure after a possible remote mutation, or surviving processes. | Preserve the last known state and the uncertainty; do not convert it to success or failure. |

Rules:

- A clean test timeout with a completed inventory is `failed` with cause `infra_failed`, not `unknown`. A timeout whose cleanup or inventory did not complete is `unknown`.
- A local receipt or ledger write error must not turn a succeeded remote call into a failure. Report the receipt error separately and preserve the remote result.
- Never retry a mutating command after `unknown` until the original effect is reconciled. The first action is to query the original run/job and inventory the target, not to re-run the command.
- If the effect remains unknown, mark the lease `unknown`, stop the dependent expansion, and escalate. Do not blend `unknown` into a completion-rate denominator.
- Preserve failed and unknown outcomes in the report. A later success on a different run does not retroactively close the earlier unknown.

## Delivery note

If the verified patch is delivered as a pull request, derive the title and body from the actual diff and commit range. Do not describe a platform, feature, or verification result that the diff does not support. A successful PATCH is not proof of the write: read the PR back and compare the title/body. Do not send test or debug comments to the target repository. Mark incomplete acceptance explicitly rather than presenting a draft as a finished validation.

## Stop and escalate conditions

Stop before the next step when:

- the target commit, container, or CANN environment is not the declared one;
- the applied diff has an unexplained file or an unrecorded dirty state;
- a fast reproducer fails, or the expected symbol/fallback is not observed;
- the outer timeout would be less than the sum of inner budgets plus cleanup grace;
- the external effect is `unknown` and the original run cannot be reconciled;
- the lease for the device, port, worktree, or `/tmp` root is missing or ambiguous.

## Scope and evidence boundary

This SOP is marked `reproduced_empirical` because the recorded live NPU session repeatedly used the loop: isolated worktree edits, container syncs, fast operator/one-layer checks, long parallel builds, service startup, profiling, and explicit refusal to expand while an earlier failure remained unresolved. The transcript is a redacted digest, not raw logs and not an independently controlled trial. The timeout and terminal-state contracts come from the MindIE bench and plugin software contracts cited in the provenance record.

The unit prescribes a verification procedure. It does not establish a speedup, a universal build time, or a success rate for remote NPU development.
