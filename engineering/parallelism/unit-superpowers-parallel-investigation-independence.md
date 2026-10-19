---
category: sop
conditions:
  repository: obra/superpowers
  revision: b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37
content_digest: sha256:c6b8b13788257f439d2617f2d19dffb9c873f124af383e23edcc10cab909be22
domain_path: engineering/parallelism
evidence_strength: single_observation
id: unit-superpowers-parallel-investigation-independence
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/unit-superpowers-parallel-investigation-independence-review.json
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L2-L3
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L10-L14
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L18-L45
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L49-L89
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L98-L109
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L126-L131
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L151-L182
  source_digest: 75a09478e9f1b5ed0be52574284e18169f600d4d689cfb15ed454dc6e7568485
  source_locator: docs/evidence/unit-superpowers-parallel-investigation-independence-review.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: Partition parallel investigations by independence and shared resources
triggers:
  error_signatures: []
  keywords:
  - superpowers
  - partition
  - parallel
  - investigations
  - by
  - independence
  - and
  - shared
  - resources
---
The Superpowers document first separates unrelated problems from coupled failures, then checks whether shared state makes concurrent work interfere. Its task brief gives each investigator a narrow scope, goal, edit constraints and required return summary. After workers return, the coordinator reviews findings, checks overlapping edits, verifies integration and spot-checks the changes. This is a prescriptive coordination procedure, not a scheduler implementation. The text asks for isolated context but provides no isolation primitive, concurrency budget, timeout, cancellation or conflict-resolution mechanism. The same-file or same-resource exclusion is more actionable than assigning one agent to every failing test file.

Reviewed limits and corrections:

- The graph explicitly routes related failures to one investigator and shared-state cases to sequential agents. It has no negative edge from the initial multiple-failures node; do not invent an executable decision algorithm.
- The frontmatter's 2+ independent tasks and the example condition of 3+ failing test files describe different units of work. Preserve the wording without asserting a proven numerical contradiction.
- The listed review-and-integrate steps do not demonstrate a transactional merge or rollback protocol. Summaries and nonoverlapping filenames alone do not establish semantic independence.
- The narrative's zero conflicts and '3 problems in time of1' provide no timing samples, serial control, token cost or failure-rate measurement. They cannot support an empirical speedup.
- The source's full-suite instruction is reported as its recommendation; this project uses focused inner-loop checks and appropriate integration checks under its own policy.

Adoption boundary: Use as a source-backed design reference for defining ownership and a return contract. Independence must be established from actual dependencies and shared resources; this knowledge is not permission to increase concurrency or run a full suite for every change.

Evidence scope: The complete four admitted source files across this three-record packet were read at the pinned Git revision. Scope is prescriptive text interpretation; no native tool execution or efficacy test was performed.

This unit is a source-derived inspection procedure at the exact repository revision in conditions. It does not establish a reproduced hardware outcome or measured performance benefit.

Pinned source ranges:

- [skills/dispatching-parallel-agents/SKILL.md:2-3](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L2-L3)
- [skills/dispatching-parallel-agents/SKILL.md:10-14](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L10-L14)
- [skills/dispatching-parallel-agents/SKILL.md:18-45](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L18-L45)
- [skills/dispatching-parallel-agents/SKILL.md:49-89](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L49-L89)
- [skills/dispatching-parallel-agents/SKILL.md:98-109](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L98-L109)
- [skills/dispatching-parallel-agents/SKILL.md:126-131](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L126-L131)
- [skills/dispatching-parallel-agents/SKILL.md:151-182](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/dispatching-parallel-agents/SKILL.md#L151-L182)
