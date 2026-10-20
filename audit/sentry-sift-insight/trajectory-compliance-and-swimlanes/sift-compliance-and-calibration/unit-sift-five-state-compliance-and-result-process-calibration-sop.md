---
category: sop
conditions:
  audit_engine: cannbot-sentry/packages/sift
  repository: cann/cannbot-sentry
  revision: c894c239444bce00cea23679de074df1fc4ed51d
  workload:
  - Post-hoc agent trajectory skill-compliance auditing, anti-fabrication verification,
    and outcome calibration
created_at: '2026-10-21'
domain_path: audit/sentry-sift-insight/trajectory-compliance-and-swimlanes/sift-compliance-and-calibration
estimate_tokens: 620
evidence_strength: software_contract
id: unit-sift-five-state-compliance-and-result-process-calibration-sop
inherits_from:
- unit-cannbot-sentry-sift-insight-cpx-audit-and-wire-redaction-workflow
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/sift/README.md#L1-L220
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/sift/sift/core/models.py#L1-L220
  promotion_status: verified
  source_locator: cann/cannbot-sentry@c894c239444bce00cea23679de074df1fc4ed51d:packages/sift/README.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Audits agent trajectories with sift across 6 skill instruction categories,
  5 compliance states, 5 result-vs-process calibration verdicts (catching HIGH_SCORE_SUSPECT),
  and 5 file-provenance dimensions.
tier: L2
title: sift Six Instruction Categories, Five-State Compliance Verdicts, and Result-vs-Process
  Calibration SOP
triggers:
  error_signatures:
  - HIGH_SCORE_SUSPECT
  - ANTI_FABRICATION
  - pass_rate_programmatic
  keywords:
  - sift compliance audit
  - five-state calibration
  - analyze_provenance five file dimensions
  - BlindComparator
updated_at: '2026-10-21'
---
# sift Six Instruction Categories, Five-State Compliance Verdicts, and Result-vs-Process Calibration SOP

## Core Principle

Evaluating an agent skill solely by benchmark outcome (`outcome_score`) masks **reward hacking, lucky shortcuts, and fabricated verification claims**. `sift` (`cannbot-sentry/packages/sift`) decomposes a skill's `SKILL.md` into atomic instructions across **6 categories**, evaluates each instruction against the recorded trajectory to produce a **5-state compliance verdict** (reporting `pass_rate_programmatic` separately from `pass_rate_llm`), and cross-calibrates process compliance against task outcome into a **5-state calibration matrix**.

## Step 1: Six Skill Instruction Categories & Dual-Channel Verification

`sift` extracts testable rules from `SKILL.md` into six categories:
1. `HEADING_STEP`: Mandatory workflow stages that must be executed in order.
2. `TOOL_REQUIREMENT`: Specific CLI tools, scripts, or MCP calls that must be invoked (`msprof`, `mssanitizer`, `run_checks.py`).
3. `CONSTRAINT`: Negative prohibitions (e.g., "never modify test files", "never use `torch` compute in wrapper").
4. `ANTI_FABRICATION`: Integrity rules requiring claims in the final response to be backed by actual tool stdout in the session log.
5. `OUTPUT_FORMAT`: Required report sections, JSON schemas, or artifact paths.
6. `REFERENCE_USAGE`: Conditional rules requiring the agent to `Read` specific `references/*.md` files when triggered.

**Five-State Per-Instruction Verdict**:
- `PASS`: Deterministic programmatic rule (or LLM judge) confirmed compliance in the trajectory.
- `FAIL`: Trajectory violated the constraint or skipped a required step/tool.
- `N/A`: Conditional trigger was not activated in this task.
- `UNRESOLVED`: Evidence is truncated or missing from the log.
- `INDETERMINATE`: Programmatic matcher could not decide and no LLM judge was enabled.

> Always track `pass_rate_programmatic` as the primary hard gate; use `pass_rate_llm` (and `BlindComparator` A/B judging) only for semantic quality rules that cannot be expressed as deterministic trajectory predicates.

## Step 2: Five-State Result-vs-Process Calibration Matrix

Cross-tabulate the task's external `outcome_passed` (benchmark pass / score threshold) against `process_compliant` (`sift` compliance threshold):

| Calibration State | Outcome Passed? | Process Compliant? | Engineering Diagnosis & Action |
| --- | --- | --- | --- |
| **`TRUSTED`** | Yes | Yes | Validated skill execution; eligible for knowledge distillation |
| **`HIGH_SCORE_SUSPECT`** | **Yes** | **No** | **High-Risk Shortcut / Reward Hacking / Fabrication**: Agent bypassed mandatory verification or violated constraints yet passed the test; quarantine trajectory and audit for test weakness |
| **`FIXABLE`** | No | No | Agent ignored the skill's SOP and failed; strengthen skill trigger/enforcement prompt |
| **`SKILL_FAILED`** | No | Yes | Agent followed every instruction in `SKILL.md` faithfully (`PASS`), yet the task still failed; **the skill's technical content itself is flawed or incomplete** and must be updated |
| **`PROCESS_UNKNOWN`** | Any | Unknown | Trajectory log incomplete or unparseable |

## Step 3: Five-Dimension File Provenance Audit (`analyze_provenance`)

Run `analyze_provenance` on the trajectory's tool calls (`Read`, `Write`, `Edit`, `Bash`) to classify every workspace file across five dimensions: (1) `read_before_write` (whether the agent inspected existing code before overwriting), (2) `verified_after_edit` (whether a test/build ran after the final edit to that file), (3) `thrash_count` (repeated revert/re-edit cycles on the same region), (4) `untouched_reference_reads`, and (5) `unverified_final_writes` (any modified production file with zero subsequent test execution is flagged as an anti-fabrication violation).
