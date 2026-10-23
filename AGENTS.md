# Agent-Assisted Engineering Domain Boundaries (`domain: "agent-engineering"`)

This repository (`knowledge-agent-engineering`) stores
agent-assisted engineering knowledge units and source-verification evidence
under the canonical domain `agent-engineering`. Runtime engine code lives
exclusively in `mindie-agent/knowledge`.

## Operating Rules

1. **Domain Scope & External Skill Boundary**: Keep agent workflow patterns
   (review git-range scoping, independent failure-domain parallel investigation,
   claim-to-evidence scope discipline) in `agent-engineering`. Treat external
   Skills (`obra/superpowers`, `cannbot`, etc.) strictly as research materials
   backed by narrow source citations; never copy external blanket mandates (such
   as running full test suites on every turn) into MindIE Agent's own operating
   rules, and never claim runtime speedups from static workflow text.
2. **Unit Protocol & Index Rebuild**: Every unit must validate as
   `mindie-graded-unit/1` with a path-independent `id`, monotonic `revision`,
   exact `content_digest`, explicit `conditions`, and line-level
   `provenance.source_locator` / `evidence_refs`. Rebuild
   `index/routing-manifest.json` and `index/shard-topology.json` via the graded
   indexer after any unit change.
3. **Domain Isolation & Three-Valued Conditions**: Bind source-derived units to
   the exact inspected `repository` and `revision` (e.g., `obra/superpowers` @
   `b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37`). Do not import operator kernels
   (`operators`), engine internals (`infer-core` / `infer-distributed`), or
   `vllm-ascend` adapter corpus files into this repository, and reject
   foreign-domain `mindie://<other-domain>/...` URIs on `explain` and `feedback`.
4. **Publication Gate**: [`./publication-contract.json`](./publication-contract.json)
   (`mindie-publication-contract/2`) and thin maintenance entry points under
   `scripts/` define the validator pin and local checks; run
   `python3 scripts/build_routing_index.py --root . --check` after any unit or
   index edit.
