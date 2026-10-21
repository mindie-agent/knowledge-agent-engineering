---
schema: mindie-graded-unit/1
id: unit-msverl-regression-triage-and-issue-faq-synthesis-workflow
revision: 1
title: "Daily CI Regression Triage (msverl) and Three-Agent Closed-Issue FAQ Synthesis Workflow"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: engineering/ci-triage/msverl-and-faq/log-tail-commit-ranking-and-issue-faq-pipeline
inherits_from:
  - unit-superpowers-parallel-investigation-independence
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: engineering
triggers:
  error_signatures:
    - "mean abs diff:"
    - "error, please check log"
    - "accuracy_regression"
    - "train_error"
  keywords:
    - "msverl-daily-regression-triage"
    - "vllm-ascend-faq-generator"
    - "log-tail-extraction"
    - "commit-ranking"
    - "issue-classification"
provenance:
  source_type: source_contract
  source_locator: Ascend/agent-skills@942f0138aa9c78467507babb09aa9c64c5f534dc:skills/msverl-daily-regression-triage/SKILL.md
  evidence_refs:
    - https://github.com/Ascend/agent-skills/blob/942f0138aa9c78467507babb09aa9c64c5f534dc/skills/msverl-daily-regression-triage/SKILL.md#L1-L63
    - https://github.com/Ascend/agent-skills/blob/942f0138aa9c78467507babb09aa9c64c5f534dc/skills/vLLM-ascend_FAQ_Generator/SKILL.md#L1-L135
  supersedes: []
---

## Falsifiable Claim

When triaging daily `verl + MindSpeed` RL training regressions or distilling hundreds of closed `vllm-ascend` GitHub issues into reusable troubleshooting FAQs, loading full multi-gigabyte training logs or raw issue streams into a single agent context exhausts token budgets and hallucinates culprit commits. Enforcing fail-fast log-tail triage with top-3 commit diff ranking (`msverl-daily-regression-triage`) and a sequential three-agent `Collector -> Classifier -> FAQ Writer` pipeline (`vLLM-ascend_FAQ_Generator`) bounds token cost while producing high-precision root-cause attribution.

## Part 1: Cost-Controlled Daily Regression Triage (`msverl-daily-regression-triage`)

### 1. Deterministic Hard-Stop Classification (`parse_result_log.py`)
Read the lightweight baseline comparison log (`msverl.log`) **first** before touching daily training logs or Git repositories:
- **`mean abs diff: 0`**: Classify as `pass` and **stop immediately** (zero log parsing or repo syncing).
- **`mean abs diff: <non-zero>`**: Classify as `accuracy_regression` and capture the reward lists and diff magnitude as primary evidence.
- **`error, please check log`**: Classify as `train_error`.
- **Ambiguous / Missing Markers**: Classify as `unknown` and report missing evidence immediately without running expensive repo scans.

### 2. Log-Tail Extraction & Two-Repo Suspect Commit Ranking
1. **Tail-First Log Extraction (`extract_failure_tail.py`)**:
   - On `train_error`, extract only the final high-signal traceback / `ERROR` block from `msverl_YYYYMMDD.log`. **Never load the full training log into context** unless tail extraction fails twice.
2. **Time-Window Commit Collection (`sync_repos.py` + `list_recent_commits.py`)**:
   - Query `verl` (`main`) and `MindSpeed` (`master`) commits from the previous day `00:00:00` up to task execution time.
3. **Two-Stage Commit Ranking (`rank_candidate_commits.py`)**:
   - Rank candidate commits using **commit titles and touched file paths first** against the error signature or accuracy drift module.
   - Read full `git diff` patches **only for the top 3 ranked candidates per repository**, and output a structured report (`status`, `time_window`, `evidence_summary`, `candidate_repo`, `candidate_commits`, `confidence: high|medium|low`, `fix_direction`).

## Part 2: Three-Agent Closed-Issue to Debug-FAQ Pipeline (`vLLM-ascend_FAQ_Generator`)

To convert closed `vllm-project/vllm-ascend` issues into structured agent/developer FAQs without context overflow, execute three specialized agents sequentially:

1. **Agent 1 — Issue Collector (`raw_issues/issue_<num>.md`)**:
   - Queries closed issues (`is:issue state:closed`), filters out noise, and extracts four mandatory fields per issue: (a) problem background, (b) exact error traceback & launch script, (c) verified root cause & fix (code PR, config flag, or env fix), and (d) `vllm` / `vllm-ascend` version matrix.
2. **Agent 2 — Issue Classifier (6-Bucket Taxonomy + Summary Report)**:
   - Classifies each normalized `issue_<num>.md` into one of six canonical directories and generates a distribution summary report:
     - `installation_setup` (CANN/driver/wheel/container setup)
     - `model_compatibility` (architecture/weight/tokenizer support)
     - `performance_tuning` (TTFT/TPOT/ACLGraph/memory optimization)
     - `api_usage` (OpenAI server/CLI/sampling parameters)
     - `hardware_adapter` (910B/910C/310P SoC & HCCL device constraints)
     - `dependency_conflict` (`torch-npu`, `triton-ascend`, C++ ABI mismatches)
3. **Agent 3 — FAQ Writer (`vLLM_ascend_debug_FAQ.md`)**:
   - Synthesizes recurring patterns within each category into actionable FAQ entries structured as: **Symptom / Error Signature** $\rightarrow$ **Root Causes** $\rightarrow$ **Step-by-Step Resolution** $\rightarrow$ **Prevention Best Practices**.
