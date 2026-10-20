---
category: workflow
conditions:
  audit_scopes:
  - skill
  - agent
  - root
  repository: cann/cannbot-sentry
  revision: c894c239444bce00cea23679de074df1fc4ed51d
domain_path: engineering/audit-observability
evidence_strength: software_contract
id: unit-cannbot-sentry-sift-insight-cpx-audit-and-wire-redaction-workflow
inherits_from:
- unit-multi-agent-npu-lease-and-container-isolation-axioms
- unit-superpowers-claim-evidence-scope
provenance:
  evidence_refs:
  - docs/evidence/cannbot-sentry-bench-and-harness-provenance.json
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/sift/README.md#L5-L250
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/insight/README.md#L7-L32
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/cpx/README.md#L5-L213
  source_digest: e79d1051fe52eaeac72092e8d3f587ee6dca75b2bb6a1613ff5fbf3e4c378182
  source_locator: docs/evidence/cannbot-sentry-bench-and-harness-provenance.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L3
title: Sentry dual-verification compliance audit (sift), wire-level single-chokepoint credential redaction (cpx), and session observability (insight) workflow
triggers:
  error_signatures:
  - High outcome score with hard process compliance violation in calibration
  - Unredacted API key or Bearer token leaked into session capture JSONL
  - Trigger probe recall inflated by lexical copying of skill description
  keywords:
  - cannbot-sentry
  - sift
  - cannbot-insight
  - cpx
  - compliance audit
  - calibration
  - analyze_provenance
  - dispatchEmit
  - wire capture
---
At the pinned `cann/cannbot-sentry` revision, `cpx` (`packages/cpx`), `cannbot-insight` (`packages/insight`), and `sift` (`packages/sift`) form a closed-loop agent engineering telemetry and compliance-audit stack: `cpx` captures wire-level LLM traffic with single-chokepoint credential scrubbing, `insight` attributes active vs wall-clock latency and multi-generation DAG runs, and `sift` reconciles outcome quality against process compliance (`packages/sift/README.md:5-250`, `packages/insight/README.md:7-32`, `packages/cpx/README.md:5-213`).

## 1. Wire-Level Verbatim Capture & Single-Chokepoint Redaction (`cpx`, `packages/cpx/README.md:5-213`)

1. **Zero-Residue Per-Session Injection (`lines 18-27`)**:
   - `cpx <agent-cmd>` injects local proxy routing only into the spawned child process environment (via `--settings` for Claude Code, `OPENCODE_CONFIG_CONTENT` for OpenCode/CANNBot, or `-c` for Codex) without mutating global user configuration files, reassembling streaming SSE chunks across Anthropic Messages, OpenAI Chat Completions, and OpenAI Responses APIs into `$HOME/.cannbot-insight/proxy/cpx-<sid>.jsonl` and `<sid>/subagents/`.
2. **Single-Chokepoint Credential Scrubbing (`dispatchEmit` / `redactor.ts`, `lines 25, 165-168`)**:
   - Every captured record passes through a single write chokepoint (`dispatchEmit()`) before touching disk.
   - `redactor.ts` combines exact structural key-name matching with vendor token-shape regexes (covering echoed `env` dumps, pasted configs, URL query parameters, and `Bearer` headers), masking secrets to `first4...last4` so vendor identity remains debuggable while raw credentials never reach disk. On session exit, live `.jsonl` files are compressed at rest into deterministic `.jsonl.7z` (LZMA2) archives.

## 2. Dual-Verification Evaluation, Multi-Scope Compliance Audit, and 5-State Calibration (`sift`, `packages/sift/README.md:5-250`)

1. **Separate Programmatic vs LLM Scoring (`sift eval` & `sift gate`, `lines 9-14, 149-201`)**:
   - `sift eval` reports deterministic programmatic assertions (file existence, AST/content invariants—zero LLM noise) separately from LLM rubric grading. `eval` is a pure data producer (`benchmark.json`); CI pass/fail decisions are computed post-hoc by `sift gate` (`0=PASS`, `1=FAIL`, `2=N/A`) across four axes: objective programmatic regression guard (always on) plus opt-in `--min-win-rate`, `--must-pass`, and `--baseline` drift verification.
2. **Multi-Scope Transcript Compliance & File-Effect Provenance (`sift audit`, `lines 11-15, 202-229`)**:
   - `sift audit <decl_path> --transcript <jsonl|json> --kind <skill|agent|root>` reconciles real execution traces against declared instructions at three distinct scopes: invoked `skill` segments, dispatched `agent` sub-sessions, or top-level `root` workflow orchestrators.
   - Outputs per-instruction verdicts alongside an instruction **coverage view** (`covered` / `uncovered` / `unknown` sections of `SKILL.md`) and a domain-tool **boundary view** (domain tools invoked in the trace vs declared tools).
   - When consuming `sift-records` v2 JSON bundles, deterministic `analyze_provenance(digest)` evaluates five file-effect dimensions—`evidence health`, `attribution integrity`, `file churn`, `actor ownership`, and `verification closure`—into `AuditReport.provenance`.
3. **Result $\times$ Process Calibration & Anti-Leakage Trigger Probes (`lines 12, 230-249`)**:
   - **5-State Calibration**: Cross-checks outcome score against process compliance; if an agent produces a correct final answer while violating mandatory process constraints (e.g., skipping required verification or mutating forbidden files), calibration flags the high score as suspect rather than `PASS`.
   - **Trigger Probes (`sift trigger`)**: Tests whether an agent naturally activates a skill from its `description` alone using anti-leakage probes (preventing verbatim phrase copying that artificially inflates recall), stratified by lexical vs semantic phrasing and short/medium/long prompt lengths.

## 3. Session Wall-Clock Clustering & Multi-Generation Rollback DAGs (`cannbot-insight`, `packages/insight/README.md:7-32`)

- **Active vs Wall-Clock Attribution**: Splits session wall time into `active + inactive = wall` using event-clustering semantics where inter-event gaps $> 5\text{ minutes}$ count as inactive **unless** an upstream model API call is actively in flight.
- **Six-Dimension Trajectory Scorecard**: Evaluates sessions on `completion` (with veto power), `trust`, `quality`, `efficiency`, `redundancy`, and `autonomy`.
- **Multi-Generation Workflow Run DAG**: Imports workflow bundles (`workflow.json` + `sessions/`) into an isolated Runs view rendering rollback generation bands, per-node retry history, verification verdicts, and generator subgraphs.
