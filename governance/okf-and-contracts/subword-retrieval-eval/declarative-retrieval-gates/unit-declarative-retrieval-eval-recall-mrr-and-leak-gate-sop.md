---
category: sop
conditions:
  eval_harness: evals/run_retrieval.py & mindie_knowledge/graded/eval_harness.py
  repository: cann/cannbot-knowledge
  revision: be5882ef29fa9943305085652d2025a038ba491c
  workload:
  - Benchmarking knowledge routing precision, recall, MRR, and cross-domain leak prevention
created_at: '2026-10-21'
domain_path: governance/okf-and-contracts/subword-retrieval-eval/declarative-retrieval-gates
estimate_tokens: 600
evidence_strength: software_contract
id: unit-declarative-retrieval-eval-recall-mrr-and-leak-gate-sop
inherits_from:
- unit-ascend-identifier-subword-tokenization-and-accessor-strip-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/evals/run_retrieval.py#L1-L260
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/evals/retrieval_cases.yaml#L1-L120
  promotion_status: verified
  source_locator: cann/cannbot-knowledge@be5882ef29fa9943305085652d2025a038ba491c:evals/run_retrieval.py
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Runs declarative retrieval evaluations (retrieval_cases.yaml and graded eval_harness),
  separating hard zero-tolerance structural/leak gates from aggregate Recall-at-K,
  MRR, and foundation baseline gates.
tier: L2
title: 'Declarative Retrieval Evaluation Harness: Recall-at-K, MRR, expected_any Groups,
  and Zero-Leak Gate SOP'
triggers:
  error_signatures:
  - forbidden_leak_rate
  - foundation_baseline_rate
  - expected_any
  keywords:
  - retrieval_cases.yaml
  - run_retrieval.py
  - Recall-at-K MRR gate
  - out_of_scope leak
updated_at: '2026-10-21'
---
# Declarative Retrieval Evaluation Harness: Recall-at-K, MRR, expected_any Groups, and Zero-Leak Gate SOP

## Core Principle

Every modification to knowledge card triggers, summaries, domain sharding, or BM25/subword tokenization must be validated against a declarative retrieval benchmark (`evals/retrieval_cases.yaml` via `evals/run_retrieval.py` or `mindie_knowledge.graded.eval_harness`). Retrieval quality governance enforces a strict two-tier gate: **hard zero-tolerance structural/leak invariants** (which fail the build even without `--quality-gate`) and **aggregate statistical ranking gates** (`Recall-at-K`, `MRR`, and `foundation_baseline_rate`).

## Step 1: Authoring Declarative Retrieval Cases (`retrieval_cases.yaml`)

Each test case declares a realistic user/agent query, optional domain/platform filters, and explicit positive/negative targets:
- **`expected` (Conjunctive Must-Hit List)**: Every target path or unit ID in `expected` must appear within the top-$K$ returned results.
- **`expected_any` (Disjunctive Equivalence Groups)**: Each group in `expected_any` is a list of acceptable alternative cards; hitting **at least one** card in the group satisfies that group.
  $$\text{Recall}_{K} = \frac{|\text{hit}(E_{\text{all}})| + |\text{hit\_groups}(E_{\text{any}})|}{|E_{\text{all}}| + |E_{\text{any}}|}$$
- **`forbidden` / `not_expected` (Negative Isolation Targets)**: Cards from conflicting architectures (e.g., Ascend 910B cards when querying an Ascend 950 RegBase-only instruction, or deprecated cards) that **must not** appear in the top-$K$ results.

## Step 2: Hard Zero-Tolerance Structural & Leak Gates (Always Active)

Regardless of aggregate `Recall-at-K`, the evaluation fails immediately (`exit 1`) if **any single case** violates these four invariants:
1. **`leaked == 0` (`forbidden_leak_rate == 0.0`)**: Zero `forbidden` or `deprecated`/`superseded` units may appear in the returned top-$K$ slice.
2. **`out_of_scope == 0`**: When a query specifies a `domain` or `platform` filter, zero returned results may fall outside the requested filter scope.
3. **`missing_fields == 0`**: Every returned entry must contain all required routing metadata fields (`id`, `title`, `domain_path`, `tier`, `summary`).
4. **`count_mismatch == 0`**: Returned result count must respect `max_results` ($\le K$) and token budget caps (`max_tokens`).

## Step 3: Aggregate Statistical Quality Gates (`--quality-gate`)

When running CI or pre-release validation (`python3 evals/run_retrieval.py --quality-gate`), enforce:
- **`Recall-at-K >= min_recall_at_k`** (e.g., `Recall-at-5 >= 0.85`): Measures coverage of required domain units.
- **`MRR >= min_mrr`** (Mean Reciprocal Rank $\frac{1}{|Q|}\sum_{q} \frac{1}{\text{rank}_1(q)} \ge 0.70$): Ensures the primary target card ranks near position 1 rather than buried at position $K$.
- **Foundation Closure (`foundation_baseline_rate == 1.0`)**: In graded knowledge packs, every returned `L2`/`L3`/`L4` unit must automatically pull its transitive `L1` `inherits_from` axiom closure into `foundation_units` without missing parent references.
