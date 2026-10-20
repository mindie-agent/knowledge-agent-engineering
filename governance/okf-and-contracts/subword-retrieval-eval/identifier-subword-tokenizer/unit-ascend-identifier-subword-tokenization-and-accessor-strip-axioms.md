---
category: axiom
conditions:
  component: knowledge-query BM25/TF-IDF index
  repository: cann/cannbot-knowledge
  revision: be5882ef29fa9943305085652d2025a038ba491c
  tokenizer_mode: TOKENIZER_SUBWORD
created_at: '2026-10-21'
domain_path: governance/okf-and-contracts/subword-retrieval-eval/identifier-subword-tokenizer
estimate_tokens: 560
evidence_strength: software_contract
id: unit-ascend-identifier-subword-tokenization-and-accessor-strip-axioms
inherits_from:
- unit-superpowers-claim-evidence-scope
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/evals/test_subword_tokenizer.py#L1-L140
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/.agents/skills/knowledge-query/scripts/retrieval/index.py#L1-L180
  promotion_status: verified
  source_locator: cann/cannbot-knowledge@be5882ef29fa9943305085652d2025a038ba491c:evals/test_subword_tokenizer.py
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: 'Defines the identifier subword tokenization axioms for technical Ascend
  C/CANN retrieval: preserving compound tokens while emitting CamelCase/snake_case
  subwords and stripping accessor prefixes only on compound splits.'
tier: L1
title: Ascend Identifier Subword Tokenization (CamelCase/snake_case Split) and Accessor
  Prefix Stripping Axioms
triggers:
  error_signatures:
  - TOKENIZER_SUBWORD
  - TOKENIZER_V3
  - test_subword_tokenizer.py
  keywords:
  - subword tokenization
  - CamelCase split
  - accessor prefix stripping
  - DataCopyPad subwords
updated_at: '2026-10-21'
---
# Ascend Identifier Subword Tokenization (CamelCase/snake_case Split) and Accessor Prefix Stripping Axioms

## Core Invariant

In Ascend C and CANN operator retrieval, users query natural-language concepts (`"copy with padding"`, `"swi glu max"`, `"l0c capacity"`) while knowledge cards index compound C++/Python identifiers (`DataCopyPad`, `get_swi_glu_max_min`, `GetCoreMemSize`). Whole-identifier tokenizers (`TOKENIZER_V3`) suffer zero lexical recall whenever a query uses separated subwords or omits getter/setter prefixes. The subword tokenizer (`TOKENIZER_SUBWORD`) resolves this by **emitting both the normalized full identifier and its filtered constituent subwords** without inflating term frequency on unsplit words.

## Axiom 1: Dual Emission on Compound Identifiers Only

Given an identifier token $T$:
1. **Always emit the lowercased full token** $t_{\text{full}} = \text{lower}(T)$ (e.g., `DataCopyPad` $\to$ `datacopypad`, `rms_norm` $\to$ `rms_norm`).
2. **Boundary Splitting (`CamelCase`, `PascalCase`, `ACRONYMWord`, `snake_case`)**:
   - Split at underscore boundaries (`_`) and case transitions (`[a-z0-9][A-Z]` and `[A-Z]+[A-Z][a-z]`).
   - Examples:
     - `DataCopyPad` $\to$ full `datacopypad` + subwords `data`, `copy`, `pad` (`data` filtered if in stopword list).
     - `RMSNorm` $\to$ full `rmsnorm` + subwords `rms`, `norm`.
     - `rms_norm` $\to$ full `rms_norm` + subwords `rms`, `norm`.
3. **No Double-Counting for Unsplit Tokens**:
   - If a token has **only 1 piece** after boundary splitting (e.g., `matmul`, `vector`, `getting`), emit it **once** (`[t_full]`). Never emit a duplicate subword copy of an unsplit word, which would distort BM25 term-frequency (`tf`) saturation.

## Axiom 2: Conditional Accessor Prefix Stripping (`get_`, `set_`, `is_`, `has_`)

C++ and Python APIs heavily prefix identifiers with boilerplate accessors (`GetBlockNum`, `SetFlag`, `get_swi_glu_max_min`, `is_contiguous`).
1. **Strip Only From Multi-Piece Split Subwords**:
   - When an identifier splits into $\ge 2$ subwords, drop leading accessor/stopword pieces (`get`, `set`, `is`, `has`, `calc`, `compute`) from the emitted subword list while **retaining the full compound token**.
   - Example: `get_swi_glu_max_min` emits `get_swi_glu_max_min` (exact API match) + `swi`, `glu`, `max`, `min` (so `get` does not pollute every getter's BM25 vector).
2. **Never Strip Standalone English Words**:
   - Because accessor stripping applies **only** to pieces produced by a multi-piece compound split, standalone words like `getting`, `setting`, or `get` in prose are **never** corrupted by prefix regexes.
