---
category: sop
conditions:
  operator_paradigms:
  - VV (Vector-Vector)
  - CV (Cube-Vector MIX)
  repository: cann/cannbot-knowledge
  revision: be5882ef29fa9943305085652d2025a038ba491c
  workload:
  - Auditing Ascend C operator/API knowledge cards, CV mixed-core synchronization
    rules, and agent skill packaging
created_at: '2026-10-21'
domain_path: governance/okf-and-contracts/card-schemas-and-vv-cv/vv-cv-and-skill-contracts
estimate_tokens: 590
evidence_strength: software_contract
id: unit-ascendc-vv-cv-operator-and-skill-governance-check-sop
inherits_from:
- unit-okf-v02-bundle-profiles-provenance-and-trust-tier-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/governance/specs/schemas.md#L1-L180
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/governance/specs/skills.md#L1-L140
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/plugins-community/collaborative-agent-kernel-evolution/skills/cake-code-review/references/cv_checklist.md#L1-L120
  promotion_status: verified
  source_locator: cann/cannbot-knowledge@be5882ef29fa9943305085652d2025a038ba491c:governance/specs/schemas.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Enforces the structural audit checklist separating Vector-only (VV) and Cube+Vector
  mixed (CV) Ascend C operator cards, C++ header verification, and agent skill packaging
  boundaries.
tier: L2
title: Ascend C VV vs CV Operator Card Structural Checklists, Header Verification,
  and Skill Anti-Bloat SOP
triggers:
  error_signatures:
  - source_headers path does not exist under include
  - CrossCoreSetFlag
  - CrossCoreWaitFlag
  keywords:
  - VV vs CV checklist
  - MIX_AIC MIX_AIV
  - source_headers verification
  - skill anti-bloat
updated_at: '2026-10-21'
---
# Ascend C VV vs CV Operator Card Structural Checklists, Header Verification, and Skill Anti-Bloat SOP

## Core Principle

Ascend C operator cards (`operators/`) and kernel code reviews (`cake-code-review`) must strictly separate **VV (Vector-only, `AIV`)** pipelines from **CV (Cube+Vector mixed, `MIX_AIC` + `MIX_AIV`)** pipelines. Applying a VV single-pipe template to a CV fused operator omits mandatory cross-core flag synchronization (`CrossCoreSetFlag` / `CrossCoreWaitFlag`), `SetTaskRation` core-ratio declarations, and `Fixpipe` (`L0C -> GM/UB`) data paths.

## Step 1: VV vs CV Operator Card & Code Review Checklist

| Audit Dimension | VV (Pure Vector `AIV`) Requirement | CV (Cube + Vector `MIX_AIC` / `MIX_AIV`) Requirement |
| --- | --- | --- |
| **Core Entry Guard** | Single `AIV` execution body (`KERNEL_TYPE_AIV_ONLY`) | Explicit `#ifdef __DAV_C220_CUBE__` (`g_coreType == AIC`) vs `#ifdef __DAV_C220_VEC__` (`g_coreType == AIV`) split |
| **Host Task Ratio** | `context->SetBlockDim(usedAivCores)` | `context->SetBlockDim(usedCubeCores)` paired with `SetTaskRation(2)` (e.g., `1 AIC : 2 AIV` sub-block mapping) |
| **On-Chip Queues** | `TPosition::VECIN` (`MTE2`), `VECCALC`, `VECOUT` (`MTE3`) | Cube `A1/B1` (`L1`), `A2/B2` (`L0A/L0B`), `CO1` (`L0C`) + Vector `VECIN`/`VECOUT` |
| **Cross-Core Sync** | Intra-core `TQue` `EnQue`/`DeQue` (`SetFlag`/`WaitFlag`) | Inter-core `CrossCoreSetFlag<0x2, PIPE_FIX>(flagId)` on AIC paired with `CrossCoreWaitFlag(flagId)` on AIV (range `0..7` or `0..15`) |
| **Workspace Visibility** | Single-launch UB chaining preferred; avoid GM round-trips | When AIC writes GM workspace for AIV consumption, `CrossCoreSetFlag` must use `PIPE_FIX` (after `Fixpipe` GM write completes) before AIV `MTE2` reads GM |

## Step 2: API `source_headers` Verification (`checks/headers.py`)

When authoring or reviewing an `API` card under `knowledge/cann/ascendc/apis/`:
1. Every header listed in frontmatter `source_headers:` must resolve to an existing file under the repository's reference `include/` tree (or CANN toolkit `ascendc/include/basic_api/`).
2. Template parameters, `TPosition` constraints, alignment multiples (32B UB, 512B L1), and supported `platforms` (`a2`, `a3`, `a5`) in the card body must match the header declaration without inventing nonexistent overloads.

## Step 3: Agent Skill Anti-Bloat & Separation Contract (`governance/specs/skills.md`)

1. **Knowledge vs Skill Separation**: Domain facts, API tables, and optimization catalogs belong in `knowledge/` cards (queried on demand via `knowledge-query`), **never** pasted as multi-thousand-line static annexes inside a skill's `SKILL.md`.
2. **Progressive Disclosure**: Keep `SKILL.md` focused on trigger conditions, workflow routing, and hard guardrails (`< 500` lines), placing heavy scripts under `scripts/` and checklists under `references/`.
3. **Contract Gate Execution**: Run `python3 governance/contracts/run_checks.py` before committing any card or skill change; all 7 check modules (`frontmatter`, `schemas`, `paths`, `body`, `sources`, `lifecycle`, `trust`) must report `0` errors.
