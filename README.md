# Agent-Assisted Engineering Knowledge (`agent-engineering`)

This independent local repository (`/Volumes/data/code/knowledge-agent-engineering`,
canonical domain `agent-engineering`) holds source-verified `L2`
engineering workflow units distilled from external agent-engineering materials
(review scoping, failure-domain isolation for parallel investigation, and
claim-to-evidence scope verification). Runtime indexing, query, distillation,
and synchronization code remains in `mindie-agent/knowledge`. Authoritative
validator pinning and program execution status are recorded in
[`./publication-contract.json`](./publication-contract.json) and
`/Volumes/data/temp/mindie-003-execution/execution-status.json`.

## Directory Structure & Admitted Units

- `engineering/review/unit-superpowers-review-request-git-range.md` (`L2`,
  `category: sop`, `evidence_strength: single_observation`, `revision: 1`,
  `content_digest: sha256:9c3e73d5806b3ea7fb49cfd696f7918027f163d8a4e24ca6e29ea9be2768f5cd`)
- `engineering/parallelism/unit-superpowers-parallel-investigation-independence.md`
  (`L2`, `category: sop`, `evidence_strength: single_observation`, `revision: 1`,
  `content_digest: sha256:c6b8b13788257f439d2617f2d19dffb9c873f124af383e23edcc10cab909be22`)
- `engineering/verification/unit-superpowers-claim-evidence-scope.md` (`L2`,
  `category: sop`, `evidence_strength: single_observation`, `revision: 1`,
  `content_digest: sha256:4f9b2a5eaebf13ce63e34dfe78f4c7d7761f4d48534e10663f5a116eb6fb3564`)
- `docs/evidence/reviewed-source-admission-v1.json` — Admission provenance from
  `/Volumes/data/temp/mindie-003-execution/wave4/research/editorial-review-packets/root-superpowers-editorial-review.json`.
- `index/routing-manifest.json` & `index/shard-topology.json` — Derived routing
  and shard indexes rebuilt from the Markdown units.
- `publication-contract.json`, `.github/workflows/check.yml`, & `scripts/`
  (`validate_units.py`, `rebuild_index.py`, `check_publication_contract.py`,
  `apply_feedback.py`, `run_all_checks.py`) — Local `mindie-publication-contract/2`
  declaration and thin CLI wrappers delegating to `mindie_knowledge.graded`.

## Unit Protocol, Evidence & Revision Rules

- **Schema**: Every unit conforms to `mindie-graded-unit/1` (`id`, `revision`,
  `content_digest`, `tier`, `category`, `evidence_strength`, `title`,
  `domain_path`, `inherits_from`, `conditions`, `triggers`, `provenance`).
- **Exact Source Conditions**: All three initial units are bound to
  `conditions.repository: "obra/superpowers"` and
  `conditions.revision: "b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37"`. Default
  `Store` queries match only when the exact `repository` and `revision` are
  supplied; mismatched revisions are rejected and omitted revisions remain
  `condition_unknown` (visible only when `include_unknown=True`).
- **Research Material Boundary & Revision Discipline**: External Skills are
  treated strictly as research inputs with narrow line-level citations (4 full
  source files, 18 narrow citations), not copied wholesale into MindIE Agent
  operating rules (for example, external "run full test suite on every step"
  instructions are explicitly excluded in favor of affected-scope verification),
  and carry no runtime speed or hardware performance claim. `id` remains stable
  across path moves; any substantive edit must increment `revision`, update
  `content_digest`, and rebuild `index/*.json`.

## Publication & Strict `Feed` Boundary

Validator revision and contract metadata are tracked in
[`./publication-contract.json`](./publication-contract.json), and live release
readiness is tracked in `/Volumes/data/temp/mindie-003-execution/execution-status.json`.
Although the public GitHub remote
`https://github.com/mindie-agent/knowledge-agent-engineering` has been created,
new content has not yet been pushed (`new_content_pushed: false`) and the paired
formal runtime/plugin release cutover remains pending.
