---
category: workflow
conditions:
  observability_suite: cannbot-sentry (insight + sift eval)
  repository: cann/cannbot-sentry
  revision: c894c239444bce00cea23679de074df1fc4ed51d
  workload:
  - Session swimlane time/token attribution, 6-dimension scorecard grading, and A/B
    skill ablation experiments
created_at: '2026-10-21'
domain_path: audit/sentry-sift-insight/trajectory-compliance-and-swimlanes/insight-swimlanes-and-ablation
estimate_tokens: 620
evidence_strength: software_contract
id: unit-insight-six-swimlane-attribution-and-skill-ablation-workflow
inherits_from:
- unit-cannbot-sentry-sift-insight-cpx-audit-and-wire-redaction-workflow
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/insight/package.json#L1-L83
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/sift/README.md#L100-L240
  promotion_status: verified
  source_locator: cann/cannbot-sentry@c894c239444bce00cea23679de074df1fc4ed51d:packages/insight/package.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: 'Executes session-level trajectory observability using cannbot-insight and
  sift eval: 6-swimlane attribution, active vs wall-clock gap filtering, 6-dimension
  scorecard with completion veto, and A/B skill ablation.'
tier: L3
title: cannbot-insight Six-Swimlane Phase Attribution, Six-Dimension Scorecard, and
  A/B Skill Ablation Workflow
triggers:
  error_signatures:
  - cannbot-insight six swimlanes
  - completion veto
  - with_skill vs without_skill
  keywords:
  - six swimlanes
  - active vs wall-clock gap
  - six-dimension scorecard
  - skill ablation
updated_at: '2026-10-21'
---
# cannbot-insight Six-Swimlane Phase Attribution, Six-Dimension Scorecard, and A/B Skill Ablation Workflow

## Core Principle

Optimizing agent engineering requires knowing **where tokens and active time are spent** across the development lifecycle and proving via **controlled A/B ablation** (`with_skill` vs `without_skill`, or `skill_v1` vs `skill_v2`) that a skill reduces compile/verify thrashing rather than merely adding prompt bloat.

## Stage 1: Six-Swimlane Trajectory Attribution (`cannbot-insight`)

Map every turn and tool invocation in a captured session (`.jsonl` / `.jsonl.7z`) into one of **six canonical engineering swimlanes**:

| Swimlane | Classified Tool / Action Patterns | Optimization Target |
| --- | --- | --- |
| **1. `explore`** | Codebase search (`Grep`, `Glob`), file reads (`Read`), and `knowledge-query` | Reduce redundant full-file reads via targeted subword retrieval |
| **2. `design`** | Architectural planning, tiling math calculation, and hypothesis formulation | Prevent premature coding before UB/L0 capacity constraints are checked |
| **3. `code`** | Source creation and mutation (`Write`, `Edit`, `apply_patch`) | Minimize repeated edit-revert thrashing on the same kernel file |
| **4. `compile`** | Build invocations (`cmake`, `make`, `bisheng`, `pip install`, `build.sh`) | Eliminate syntax/header/type trial-and-error loops via VV/CV checklists |
| **5. `verify`** | Functional UT/ST, `mssanitizer`, `npu-check`, and `msprof` / `cann-bench` runs | Ensure every `code`/`compile` cycle terminates in quantitative verification |
| **6. `distill`** | Post-task summary generation, rollback advice, or knowledge card extraction | Keep concise and grounded in `verify` evidence |

### Active Time vs Wall-Clock Idle Gap Filtering
- Agent sessions often pause waiting for human approval or disconnected terminals. When computing session duration, **split event clusters at idle gaps $> 5\text{ minutes}$** (`300s`) where **zero LLM API requests or tool subprocesses are in flight**.
- Never subtract long-running in-flight tool durations (`msprof op simulator`, `cann-bench` batch compile, or LLM streaming wait) as idle time.

## Stage 2: Six-Dimension Trajectory Scorecard & `completion` Veto

Grade each session across six normalized `[0, 100]` dimensions:
1. **`completion` (Hard Veto Gate)**: Whether the required deliverable/test passed. **If `completion == 0`, the composite session score is capped/vetoed regardless of how fast or clean the trajectory was.**
2. **`trust`**: Anti-fabrication and `sift` compliance score (`pass_rate_programmatic`).
3. **`quality`**: Code hygiene, absence of unverified final writes, and precision margin.
4. **`efficiency`**: Token and active-time economy relative to task complexity.
5. **`redundancy`**: Penalty for duplicate file reads, repeated identical failing bash commands, and edit thrashing.
6. **`autonomy`**: Ability to recover from compiler/runtime errors without human intervention.

## Stage 3: Controlled A/B Skill Ablation (`with_skill` vs `without_skill`)

Before promoting a new or revised skill, run paired trajectories on the same benchmark task set under identical model and hardware conditions:
- Compare $\Delta \text{completion}$, $\Delta \text{compile\_swimlane\_tokens}$, $\Delta \text{thrash\_count}$, and calibration state distribution (`TRUSTED` vs `SKILL_FAILED`).
- Any skill revision that increases total token consumption by $> 25\%$ without improving `completion` or reducing `compile`/`verify` error loops must be pruned or refactored into on-demand `knowledge/` cards.
