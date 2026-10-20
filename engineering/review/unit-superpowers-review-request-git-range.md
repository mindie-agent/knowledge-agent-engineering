---
category: sop
conditions:
  repository: obra/superpowers
  revision: b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37
content_digest: sha256:9c3e73d5806b3ea7fb49cfd696f7918027f163d8a4e24ca6e29ea9be2768f5cd
domain_path: engineering/review
evidence_strength: single_observation
id: unit-superpowers-review-request-git-range
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/unit-superpowers-review-request-git-range-review.json
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/SKILL.md#L8-L22
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/SKILL.md#L26-L47
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/SKILL.md#L77-L103
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L3-L28
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L30-L61
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L63-L108
  source_digest: a08177afb82ad96fb9459beb5b53ec0c8a446ef9a4bd585277bffcee2b72e432
  source_locator: docs/evidence/unit-superpowers-review-request-git-range-review.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: Bind a review request to a work product and explicit Git range
triggers:
  error_signatures: []
  keywords:
  - superpowers
  - bind
  - a
  - review
  - request
  - to
  - work
  - product
  - and
  - explicit
  - git
  - range
---
At the pinned Superpowers revision, the requester supplies a description, requirements and a base/head diff to a separate reviewer, then triages Critical, Important and Minor findings. The reviewer template asks for file-and-line findings that explain defect, impact and repair, followed by a reasoned merge assessment. The intended history isolation is a written workflow rule; these two documents do not implement process or context isolation. The caller's five-field list omits PLAN_REFERENCE, which is a sixth distinct literal used by the template. A caller integrating this template must resolve both plan fields and bind the Git range to the actual work being reviewed. The supplied HEAD~1 example compares committed trees and does not include uncommitted edits.

Scope and caveats:

- The extra template field is PLAN_REFERENCE; PLAN_OR_REQUIREMENTS already appears in both documents. Do not claim it is absent from the reviewer template.
- Giving feedback only on reviewed code does not forbid reading unchanged callers, schemas or dependencies. A diff is the supplied entry point, not a stated read confinement boundary.
- Treat history isolation, review cadence and severity handling as prescriptions, not as observed tool behavior or guaranteed defect prevention.
- There is no explicit fix-to-rerun or issue-tracking protocol in these two units. Do not infer remediation closure from a requested fix or a merge verdict.
- The documents' two template-path spellings are visible, but no deployed resolver failure was demonstrated. The example outputs are illustrations, not test or review results.

Adoption boundary: Useful as a review-request and finding-format design reference. Apply the current project's scope, authorization and verification rules; this source does not authorize spawning agents, changing a repository, or imposing its per-task review cadence.

Evidence scope: The complete four admitted source files across this three-record packet were read at the pinned Git revision. Scope is prescriptive text interpretation; no native tool execution or efficacy test was performed.

This unit is a source-derived inspection procedure at the exact repository revision in conditions. It does not establish a reproduced hardware outcome or measured performance benefit.

Pinned source ranges:

- [skills/requesting-code-review/SKILL.md:8-22](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/SKILL.md#L8-L22)
- [skills/requesting-code-review/SKILL.md:26-47](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/SKILL.md#L26-L47)
- [skills/requesting-code-review/SKILL.md:77-103](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/SKILL.md#L77-L103)
- [skills/requesting-code-review/code-reviewer.md:3-28](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L3-L28)
- [skills/requesting-code-review/code-reviewer.md:30-61](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L30-L61)
- [skills/requesting-code-review/code-reviewer.md:63-108](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L63-L108)
