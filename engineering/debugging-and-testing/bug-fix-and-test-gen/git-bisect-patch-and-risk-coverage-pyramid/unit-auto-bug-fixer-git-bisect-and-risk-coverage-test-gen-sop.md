---
schema: mindie-graded-unit/1
id: unit-auto-bug-fixer-git-bisect-and-risk-coverage-test-gen-sop
revision: 1
title: "Automated Bug Root-Cause Localization (E001-E004 Gates) and Risk-Coverage Unit Test Generation SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: engineering/debugging-and-testing/bug-fix-and-test-gen/git-bisect-patch-and-risk-coverage-pyramid
inherits_from:
  - unit-superpowers-claim-evidence-scope
  - unit-superpowers-review-request-git-range
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: engineering
triggers:
  error_signatures:
    - "E001"
    - "E002"
    - "E003"
    - "E004"
    - "git bisect bad"
  keywords:
    - "auto-bug-fixer"
    - "auto-develop-test-gen"
    - "git-bisect"
    - "risk-coverage"
    - "given-when-then"
    - "regression-test"
provenance:
  source_type: source_contract
  source_locator: Ascend/agent-skills@942f0138aa9c78467507babb09aa9c64c5f534dc:skills/auto-bug-fixer/SKILL.md
  evidence_refs:
    - https://github.com/Ascend/agent-skills/blob/942f0138aa9c78467507babb09aa9c64c5f534dc/skills/auto-bug-fixer/SKILL.md#L1-L140
    - https://github.com/Ascend/agent-skills/blob/942f0138aa9c78467507babb09aa9c64c5f534dc/skills/auto-develop-test-gen/SKILL.md#L1-L122
  supersedes: []
---

## Falsifiable Claim

Automated bug fixing and unit test generation fail when agents emit speculative patches without locking a reproducible root cause or write assertion-free tests that inflate line coverage ("numeric coverage") without testing failure branches ("risk coverage"). Combining the four-stage `auto-bug-fixer` pipeline (enforcing `E001–E004` fail-closed interrupt codes and `git bisect` commit verification) with the seven-step `auto-develop-test-gen` risk-coverage methodology guarantees minimal-diff root-cause fixes backed by falsifiable Given-When-Then regression tests.

## Part 1: Four-Stage Automated Bug Localization & Patch Pipeline (`auto-bug-fixer`)

### 1. Fail-Closed Stage Gates (`E001` – `E004`)
Every stage validates its preconditions and aborts immediately with a deterministic error code if unmet:
- **`E001` (Input Incomplete/Unparseable)**: Missing `error_phenomenon` or `reproduce_steps`, or corrupted traceback logs. Abort before attempting code search.
- **`E002` (Ambiguous Root Cause)**: Cannot lock the defect to an exact file, line range, or commit/PR with reproducible verification. Abort and request narrower context rather than guessing a fix.
- **`E003` (Patch Generation Failure)**: Root cause is not locked or target code context is missing.
- **`E004` (Regression Test Failure)**: Patch is unverified or target test framework (`pytest`, `gtest`, `JUnit`, `GoTest`) is unspecified.

### 2. Dual Root-Cause Localization Strategy
- **Method A — Top-Down Dataflow Trace (No PR/Commit Range Provided)**:
  Start at the exact exception frame in the stack trace, inspect input tensor/argument invariants, and trace upstream callers step-by-step until locating the first site where state diverges from contract.
- **Method B — `git bisect` Commit Localization (Change-Induced Regression)**:
  Identify the last known good commit (`<good_sha>`) and first broken commit (`<bad_sha>`):
  ```bash
  git bisect start
  git bisect bad <bad_sha>
  git bisect good <good_sha>
  ```
  Verify the isolated culprit commit by reverting it (`git revert --no-commit <culprit_sha>`) and confirming the failure disappears before authoring the minimal unified `diff` patch.

## Part 2: Risk-Coverage Unit Test Generation SOP (`auto-develop-test-gen`)

### 1. Distinguish Risk Coverage from Numeric Coverage
Never write filler tests merely to raise line coverage percentages. Prioritize untested code by **risk density**:
1. **Priority 1 (Critical)**: Core business invariants, numerical precision gates, and error/timeout handling paths.
2. **Priority 2 (High)**: Boundary conditions (empty inputs, $0$/$\text{INT\_MAX}$ dimensions, threshold edges) and historical defect hotspots.
3. **Priority 3 (Low)**: Trivial getters or low-complexity formatting branches.

### 2. Three-Tier Given-When-Then Test Construction
For every target function or class:
1. **Extract Contract**: Identify input domain, return value, state mutations, and external dependencies.
2. **Three-Tier Test Matrix**:
   - **Happy Path**: Typical valid inputs produce exact expected outputs.
   - **Boundary Path**: Minimum/maximum sizes, zero-length slices, and saturation boundaries.
   - **Exception Path**: Invalid arguments, upstream dependency failures, and timeouts raise explicit exceptions without corrupting state.
3. **Single-Behavior Assertion Rule**: Each test function follows `Given-When-Then` (`Arrange-Act-Assert`), verifies **one** core behavior against public observable outputs (never coupling to private internal implementation details), and fails before the bug fix while passing after the fix.
