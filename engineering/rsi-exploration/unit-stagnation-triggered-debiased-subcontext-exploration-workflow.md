---
category: workflow
conditions:
  native_context_isolation: required
  optimization_mode: rsi_exploration
  qualification_gates:
  - G0_compile
  - G1_hidden_input_5_category
  - G2_profiler_speedup
  - G3_contract_c4_sequential_ablation
  stagnation_threshold: 3_consecutive_unimproved_trials
  subagent_pack_schema: mindie-rsi-subagent-pack/1
domain_path: engineering/rsi-exploration
evidence_strength: software_contract
id: unit-stagnation-triggered-debiased-subcontext-exploration-workflow
inherits_from:
- unit-multi-agent-npu-lease-and-container-isolation-axioms
- unit-remote-npu-container-patch-and-long-job-verification-sop
- unit-superpowers-parallel-investigation-independence
provenance:
  evidence_refs:
  - docs/evidence/agent-engineering-npu-provenance.json
  - mindie-agent-codex@24049537746b23f9a5fc91412319a73488857a8e:plugins/mindie-agent/skills/mindie-agent/SKILL.md
  - mindie-agent-codex@24049537746b23f9a5fc91412319a73488857a8e:plugins/mindie-agent/skills/mindie-agent/references/domain-skills.md
  - mindie-agent-codex@24049537746b23f9a5fc91412319a73488857a8e:plugins/mindie-agent/skills/mindie-agent/references/diagnostic-reasoning.md
  - mindie-bench@c42493d6ff916debf2967b76adace62113b41340:AGENTS.md
  - knowledge-operators@worktree-snapshot-20261021:Reduction/unit-vectorized-bf16-fp32-accumulation-cast-fusion-mutator.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/ops-perf-evolution/AGENTS.md
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/ops-perf-evolution/agents/lingxi-evo.md
  source_digest: d36d173738cc47863d74fddd3772ae381c78503f1884822f02d16add39adda1b
  source_locator: docs/evidence/agent-engineering-npu-provenance.json
  source_type: software_contract_and_plugin_snapshot
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L3
title: Break optimization stagnation with a de-biased native subcontext
triggers:
  error_signatures:
  - stagnation
  - no improvement
  - three consecutive unimproved
  - assess_stagnation
  - conditional_avoid
  - pruned_combination_verified
  - candidate_for_sequential_pruning
  - opt_regression
  - evaluation_error
  keywords:
  - RSI
  - exploration
  - stagnation
  - de-biased
  - native subcontext
  - subagent pack
  - mindie-rsi-subagent-pack/1
  - L1 foundation
  - L2 bottleneck
  - L4 mutator
  - leave-one-out ablation
  - sequential ablation
  - hidden inputs
  - knowledge flywheel
content_digest: sha256:11c8b15cc880af33c76065f58f2996b951e638c9c063219929dd0f63236c1346
---
# Workflow

Enter this workflow only when the optimization ledger shows at least three consecutive unimproved trials, deduplicated by `run_id`, and the trials produced neither a new fact nor an identified divergent layer. If a new fact appears, continue the current route and record the fact. If a first divergence is identified, that is progress; do not force context isolation merely because the trial count reached three.

The workflow has four stages. Do not skip a stage, and do not treat a later gate as a substitute for an earlier one.

## Stage 1 — Freeze the ledger evidence

Stop varying the current L3 route. Preserve the state that produced the stagnation before changing the context.

1. Snapshot `PROGRESS.md` and `experiment-ledger.jsonl`; record a SHA-256 digest of each snapshot.
2. Record the goal, acceptance criteria, remaining budget, the verified baseline `run_id`, the best verified candidate `run_id`, the current worktree `git_commit`/`diff_digest`, the remote endpoint, the last verified environment timestamp, and every active `in_flight_runs` entry.
3. Record the active hypothesis, the factors and factor roles, the unit refs (`unit_id` and `revision`), the receipt refs (`manifest`, `profile`, `ablation`), the scope conditions (`soc`, `cann`, `dtype`, `shape_family`), and every still-relevant `conditional_avoid` with its `exp_id` and `run_id`.
4. Keep execution facts separate from hypothesis judgments. The `Run Manifest` owns `passed`, `failed`, `inconclusive`, and `cancelled`; `unknown` is reserved for uncertain external writes. The ledger owns `supported`, `refuted`, and `inconclusive`.
5. Compile errors, SSH drops, container losses, and invalid measurement windows are `inconclusive`, never domain `refuted`.
6. A local ledger or receipt write error must be reported separately. It must never overwrite a succeeded remote execution result.

The output is a frozen evidence bundle, not a rewritten history. Do not delete failed attempts and do not merge unverified edits into the verified baseline.

## Stage 2 — Build a fresh native subcontext pack

Call the knowledge exploration surface with the frozen evidence:

```text
knowledge_query(
  query=<unresolved subproblem>,
  mode="explore",
  native_subcontext=true,
  ledger_entries=[...],
  bottleneck_summary={...},
  goal=<goal and acceptance criteria>
)
```

Then enforce the isolation contract:

- Require `exploration_context.subagent_fact_pack.schema_version == "mindie-rsi-subagent-pack/1"`.
- Inspect `stripped_l3_ids`. The pack must show that prior L3 workflows were removed.
- Keep L1 hardware axioms and hard constraints, L2 bottleneck measurements, L4 mutators, and `conditional_avoid`. Do not keep prior L3 route instructions, prior conversation text, or a full-history handoff.
- If `native_subcontext` is false, the result is an in-session warning, not a real isolation claim. Do not present it as a fresh native context.
- If the pack contains prior L3 workflows, stale route instructions, or an unverifiable isolation claim, reject the pack and rebuild it.

Spawn a fresh native subagent or native subcontext seeded only with `subagent_fact_pack.prompt_brief`. Do not paste the old conversation into the new context as a shortcut. Record the pack digest, `stripped_l3_ids`, and the prompt-brief digest with the exploration `exp_id`.

The output is a new search context that cannot simply replay the old L3 trajectory.

## Stage 3 — Enforce the four-stage qualification gates

Run the gates in order. A failure at a gate is a result, not a reason to relax the gate.

### G0 — Compile

- Compile the candidate with the repository's native build path and the declared CANN/SOC environment.
- Import the module and confirm that the intended symbol or kernel is selected. A fallback composite or an old cached binary is not a pass.
- Require the build log, the selected-symbol evidence, the changed-file hashes, and the command/exit receipt.
- A compile failure rejects the implementation, not necessarily the direction. Preserve the error as a diagnostic finding.

### G1 — Hidden-input five-category generalization

- Freeze hidden inputs before running the candidate. Do not tune the candidate against them.
- Re-run the retained candidate on every frozen hidden item and on the guard workloads: aligned shape, unaligned tail, tile-boundary crossing, workspace high-water mark, graph capture/replay shape stability, and warm-server repeated call.
- Classify each item as exactly one of: `both_pass`, `opt_regression`, `both_fail`, `opt_improvement`, or `evaluation_error`.
- Only comparable `both_pass` items enter a speedup ratio. A local gain never overrides a guard regression. An `opt_regression` rejects system acceptance even if the main case is faster.
- An `evaluation_error` is not a pass and is not silently dropped from the denominator. Preserve it and explain the missing observation.

### G2 — Profiler speedup

- Measure on the leased device and the same clock domain. Record the device, topology, CANN version, dtype, shape family, graph mode, warmup count, and measurement window.
- Use synchronized measurement windows and at least two warmups. Keep case identity and hashes per sample; report per-case block envelopes rather than a pooled number that hides a regression.
- Compare the candidate against a valid baseline with the same topology and workload. A changed topology or a different input signature invalidates the comparison.
- Do not add isolated-kernel savings into an end-to-end gain. Missing durations remain unknown; hardware lower bounds are not attainable latency predictions.
- A speedup is reportable only if G0 and G1 pass. If the candidate is not demonstrated to be the code that ran, the timing is invalid.

### G3 — leave-one-out ablation sequential ablation

- Run a leave-one-out comparison of the candidate's factors against the full candidate and the baseline.
- A leave-one-out `delta <= tolerance` only sets `disposition: "candidate_for_sequential_pruning"`. It never auto-deletes the factor.
- Treat `disposition == "candidate_for_sequential_pruning"` with `pruned_combination_verified == false` as RETAIN.
- Only sequential, one-at-a-time `pruning_runs` whose final combination passes both `correctness_verified` and `performance_verified` can prune. The final combination must be verified as the final combination, not inferred from an earlier arm.
- L4 distillation requires a verified ablation receipt with a `baseline_run_id` and a `verification_run_id`. Without that receipt, the result is not an L4 transformation and must not be distilled as one.

Gate discipline:

- G0 failure stops the chain. Do not benchmark a candidate that did not compile or did not select the intended symbol.
- G1 failure stops system acceptance. Do not average a regression away.
- G2 without a valid baseline or a valid measurement window is `unknown`, not a speedup.
- G3 without a verified pruning/final-combination receipt is RETAIN, not a simplification.

## Stage 4 — Distill verified improvements into the knowledge flywheel

Distill only after the applicable gates pass. If the result is a diagnostic finding without a performance gain, record it as an L2/L3 finding; L4 admission still requires the G3 receipt.

1. Call the distillation surface with `task_state="verified_milestone"` for a finished sub-result in an ongoing task, or `"completed"` for completion. Include `related_refs` to same-topic units so the organizer can prefer revise, merge, or supersede over a duplicate.
2. Include the verified `unit_refs` (`unit_id` and `revision`), the `scope_conditions`, the `conditional_avoid`, and the receipt refs. Do not promote a hypothesis to a rule because it survived one attempt.
3. Preserve counterevidence, failed attempts, human corrections, and unresolved alternatives. A negative result with a new discriminating fact is valuable; it is not a reason to rewrite the ledger as a success.
4. Update the ledger with the exploration `exp_id`, the pack digest, the gate receipts, and the final judgment (`supported`, `refuted`, or `inconclusive`). Keep the judgment scoped to the hardware, CANN version, dtype, and shape family that were actually tested.
5. Rebuild the routing index after the unit change. Do not claim a measured speedup, a universal applicability, or a completed optimization that the receipts do not establish.

The output is a scoped improvement or a scoped negative finding in the flywheel, not an unbounded “the agent got better” claim.

## Exit criteria

The workflow ends in one of three recorded states:

- **Verified improvement.** G0-G3 pass, the final combination is verified, and the unit/ledger update is scoped to the observed conditions.
- **Refuted direction.** A gate fails for a reason that is attributable to the candidate, and the failure is preserved with its scope. The next route may use the new fact.
- **Inconclusive.** The evidence is missing, the measurement is invalid, the environment changed, or the isolation claim is unverified. Stop and escalate; do not convert this state into a pass.

In all three states, release the device/port/worktree lease only after the evidence bundle and the ledger update are durable.

## Scope and evidence boundary

This is a prescriptive L3 workflow assembled from the MindIE Agent stagnation/native-subcontext contract, the ledger and leave-one-out ablation consumer rule, the hidden-retention five-category contract, the G0-G3 qualification-ladder reference, and the CANNBot evolution-loop contracts. The recorded live NPU session supplies usage context for the failure modes that motivate the workflow, not a controlled efficacy trial.

The unit does not establish that context isolation improves optimization outcomes, that any particular candidate passes the gates, or that a measured speedup exists. It defines the evidence required before such a claim may be made.
