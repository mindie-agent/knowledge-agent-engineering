---
category: axiom
conditions:
  bundle_spec: OKF v0.2
  repository: cann/cannbot-knowledge
  revision: be5882ef29fa9943305085652d2025a038ba491c
  workload:
  - Domain knowledge bundle authoring, frontmatter schema enforcement, and lifecycle
    governance
created_at: '2026-10-21'
domain_path: governance/okf-and-contracts/card-schemas-and-vv-cv/okf-bundle-and-trust-lifecycle
estimate_tokens: 600
evidence_strength: software_contract
id: unit-okf-v02-bundle-profiles-provenance-and-trust-tier-axioms
inherits_from:
- unit-superpowers-claim-evidence-scope
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/governance/specs/okf.md#L1-L160
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/governance/specs/frontmatter.md#L1-L210
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/governance/contracts/checks/trust.py#L1-L120
  promotion_status: verified
  source_locator: cann/cannbot-knowledge@be5882ef29fa9943305085652d2025a038ba491c:governance/specs/okf.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Establishes OKF v0.2 single-bundle invariants, nine directory path profiles,
  40-hex Git commit provenance pinning, and orthogonal lifecycle status vs derived
  verification trust tiers.
tier: L1
title: OKF v0.2 Nine Path Profiles, 40-Hex Commit Provenance Pinning, and Orthogonal
  Trust Tier Axioms
triggers:
  error_signatures:
  - replaced_by is required when status is deprecated
  - machine-confirmed
  - human-reviewed
  keywords:
  - OKF v0.2
  - nine path profiles
  - orthogonal trust tier
  - 40-hex commit pinning
updated_at: '2026-10-21'
---
# OKF v0.2 Nine Path Profiles, 40-Hex Commit Provenance Pinning, and Orthogonal Trust Tier Axioms

## Core Invariant

An Open Knowledge Format (`OKF v0.2`) repository treats `knowledge/` as a **single unified bundle** governed by machine-executable contracts (`governance/contracts/run_checks.py`). Every knowledge card must satisfy three non-negotiable axioms: (1) its relative path under `knowledge/<domain>/` must map to one of the **9 canonical content profiles**, (2) every external Git source must be pinned to an **immutable 40-character hexadecimal commit SHA**, and (3) **lifecycle `status`** (`draft` / `stable` / `deprecated`) is strictly orthogonal to **derived verification trust tier** (`unverified` / `machine-confirmed` / `human-reviewed`).

## Axiom 1: Nine Canonical Path Profiles & Single-H1 Body Hygiene

Every leaf Markdown file (`*.md`, excluding `index.md` navigation hubs) must reside under one of nine profile directories, which determines its required `type` and structural contract:

| Directory Profile | Required `type` | Mandatory Body / Frontmatter Contract |
| --- | --- | --- |
| `concepts/` | `Concept` | Defines domain mental models, invariants, and architectural trade-offs |
| `apis/` | `API` | Declares `source_headers` verified against `include/` and exact function signatures |
| `examples/` | `Example` | Self-contained runnable or compilable code skeleton with input/output contract |
| `guides/` | `Guide` | Decision trees, boundary tables, and multi-scenario engineering rules |
| `operators/` | `Operator` | Ascend C VV/CV operator design, tiling layout, and pipeline dataflow |
| `optimizations/` | `Optimization` | Before/after code transformation, bottleneck condition, and measured speedup |
| `runbooks/` | `Runbook` | Step-by-step operational SOP with explicit failure branches and verification gates |
| `interoperability/` | `Interoperability` | Cross-framework migration mappings (`CUDA -> Ascend C` / `Triton -> Ascend C`) |
| `glossaries/` | `Glossary` | Canonical terminology definitions and disambiguation |

- **Body Hygiene (`checks/body.py`)**: Every leaf card must contain **exactly one H1 (`# ...`) matching `title`**, zero broken relative Markdown links, and zero collapsed single-line Markdown table rows (`| a | b | | c | d |`).

## Axiom 2: Immutable 40-Hex Commit Provenance (`checks/sources.py`)

In `sources[]`, any URL pointing to a Git hosting platform (`gitcode.com`, `github.com`, `gitee.com`) **must not** reference mutable branches (`blob/main/`, `blob/master/`, `raw/dev/`). It must pin an exact 40-character lowercase hex commit SHA (`blob/<40-hex-sha>/<path>`) so future upstream refactors never silently invalidate the card's cited evidence.

## Axiom 3: Orthogonal Lifecycle `status` vs Derived Trust Tier (`checks/lifecycle.py` & `checks/trust.py`)

Never conflate editorial lifecycle state with empirical verification strength:
1. **Editorial Lifecycle (`status`)**:
   - `draft` -> `stable` -> `deprecated`.
   - When `status: deprecated`, frontmatter **must** declare a non-empty `replaced_by` pointer to the superseding card path; active navigation `index.md` files must not route new queries to deprecated cards without marking them deprecated.
2. **Derived Verification Trust Tier (computed from `verified[]`, never hand-written as a standalone field)**:
   - **`unverified`**: `verified` list is absent or empty.
   - **`machine-confirmed`**: At least one `verified[]` entry has `by: ci/<pipeline>` or `by: tool/<runner>` with a valid hardware `env` (`platforms`) and timestamp, but no human reviewer entry.
   - **`human-reviewed`**: At least one `verified[]` entry records `by: human/<handle>` (or maintainer sign-off) alongside reproducible command/environment evidence.
