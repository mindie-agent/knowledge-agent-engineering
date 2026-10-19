---
category: sop
conditions:
  repository: obra/superpowers
  revision: b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37
content_digest: sha256:4f9b2a5eaebf13ce63e34dfe78f4c7d7761f4d48534e10663f5a116eb6fb3564
domain_path: engineering/verification
evidence_strength: single_observation
id: unit-superpowers-claim-evidence-scope
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/unit-superpowers-claim-evidence-scope-review.json
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L18-L38
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L40-L50
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L65-L74
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L78-L106
  - https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L108-L125
  source_digest: 4f54912ea0f00ffac1e0d479b58ea6a0d55461d77fe832a5db46b3405703d992
  source_locator: docs/evidence/unit-superpowers-claim-evidence-scope-review.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: Match completion claims to observable evidence and its scope
triggers:
  error_signatures: []
  keywords:
  - superpowers
  - match
  - completion
  - claims
  - to
  - observable
  - evidence
  - and
  - its
  - scope
---
The pinned verification skill describes a five-step reporting gate: identify evidence for a particular claim, run the relevant verification, read the output and exit status, compare the result with the claim, then report success or the actual failure with that evidence. Its claim table distinguishes a lint result from a build, a changed file from a fixed symptom, a passing test from requirements coverage, and an agent's statement from independently inspected changes. These are evidence-category distinctions. The source supplies no automated gate, freshness state machine, compliance measurement or proof that any one command establishes general correctness.

Reviewed limits and corrections:

- Correct the gate citations: identify29, run30, read31, verify32, failure/success branches33-34, and claim35; earlier candidates shifted these by one line.
- The document literally demands a fresh full command in the current message and says partial checks prove nothing. Report that as source wording, not a universal engineering fact: a scoped check can support a scoped claim, and unrelated repeated suites do not strengthen it.
- An observed exit0 is evidence about the command that ran, not by itself proof of complete requirements, correct environment or absence of untested defects.
- The VCS-diff requirement in delegated-work examples is useful for code changes; do not turn it into a universal completion oracle for read-only, remote or non-code tasks.
- Remove assertions about unseen earlier candidate time-pressure wording. The '24 failure memories' are unaudited narrative evidence and establish no defect-reduction, cost or productivity effect.

Adoption boundary: Reuse the distinction between a claim and the evidence required to support it. Preserve failed and unobserved outcomes, but follow current project policy for test scope and avoid repeating verification solely because a new message is being written.

Evidence scope: The complete four admitted source files across this three-record packet were read at the pinned Git revision. Scope is prescriptive text interpretation; no native tool execution or efficacy test was performed.

This unit is a source-derived inspection procedure at the exact repository revision in conditions. It does not establish a reproduced hardware outcome or measured performance benefit.

Pinned source ranges:

- [skills/verification-before-completion/SKILL.md:18-38](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L18-L38)
- [skills/verification-before-completion/SKILL.md:40-50](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L40-L50)
- [skills/verification-before-completion/SKILL.md:65-74](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L65-L74)
- [skills/verification-before-completion/SKILL.md:78-106](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L78-L106)
- [skills/verification-before-completion/SKILL.md:108-125](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/verification-before-completion/SKILL.md#L108-L125)
