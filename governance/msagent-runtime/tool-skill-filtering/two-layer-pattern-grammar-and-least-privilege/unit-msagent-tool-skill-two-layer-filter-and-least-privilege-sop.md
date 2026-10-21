---
schema: mindie-graded-unit/1
id: unit-msagent-tool-skill-two-layer-filter-and-least-privilege-sop
revision: 1
title: "MindStudio Agent (msagent) Least-Privilege Governance: Tool/Skill Pattern Grammar and Two-Layer Enforcement SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: governance/msagent-runtime/tool-skill-filtering/two-layer-pattern-grammar-and-least-privilege
inherits_from:
  - unit-okf-v02-bundle-profiles-provenance-and-trust-tier-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: governance
triggers:
  error_signatures:
    - "impl:deepagents:"
    - "use_catalog: false"
    - "AgentFactory.create"
  keywords:
    - "msagent"
    - "tool-filtering"
    - "skill-filtering"
    - "deepagents"
    - "least-privilege"
    - "model-call-middleware"
provenance:
  source_type: source_contract
  source_locator: Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:docs/zh/user_guide/agent-tool-skill-filter-rules.md
  evidence_refs:
    - https://github.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/docs/zh/user_guide/agent-tool-skill-filter-rules.md#L1-L121
    - https://github.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/AGENTS.md#L89-L138
  supersedes: []
---

## Falsifiable Claim

In `msagent` (`mindstudio-agent`, built on DeepAgents + LangGraph), filtering tools only at agent initialization allows DeepAgents default-injected runtime tools to bypass least-privilege restrictions during model invocation. `msagent` enforces a strict fail-closed pattern grammar (`<category>:<module>:<name_pattern>` for tools and `<category>:<name_pattern>` for skills) at **two independent layers**—first in `AgentFactory.create` when constructing the initial tool catalog, and second inside the `model-call middleware` prior to every LLM turn—while rejecting legacy module aliases (`file_system`, `terminal`, `execute`).

## Pattern Grammar & Fail-Closed Evaluation Semantics

### 1. Three-Segment Tool Pattern Grammar (`tools.patterns`)
Every entry in `tools.patterns` must follow `<category>:<module>:<name_pattern>` (with optional leading `!` for negative rules):
- **`category`**:
  - `impl`: Built-in implementation tools (always paired with module `deepagents`).
  - `mcp`: External Model Context Protocol tools.
- **`module`**:
  - Fixed to `deepagents` when `category == "impl"`.
  - Set to the configured MCP server identifier (e.g., `msprof-mcp`) when `category == "mcp"`.
- **`name_pattern`**: `fnmatch` wildcard pattern (`*`, `?`, `[abc]`), such as `get_tool`, `run_tool`, `fetch_skills`, `get_skill`, or `*_debug`.
- **No Legacy Aliases**: Malformed non-three-segment strings or legacy aliases (`execute`, `file_system`, `grep_search`, `terminal`) are ignored with a warning and never matched.

### 2. Two-Segment Skill Pattern Grammar (`skills.patterns`)
Every entry in `skills.patterns` must follow `<category>:<name_pattern>` (with optional leading `!`):
- **`category`**: Skill domain directory under `skills/` (`profiler`, `accuracy`, `quantizer`, `modeling`, `operator`, `benchmark`, `minos`, `speculators`, or `default`).
- **`name_pattern`**: `fnmatch` skill name pattern (e.g., `ascend-profiler-*`).

### 3. Fail-Closed Boolean Evaluation Contract
For any candidate tool or skill $x$, let $P$ be the set of positive patterns (without `!`) and $N$ be the set of negative patterns (prefixed with `!`):
$$\text{allowed}(x) \iff (\exists p \in P,\; \text{fnmatch}(x, p)) \;\land\; \neg(\exists n \in N,\; \text{fnmatch}(x, n))$$
- **Empty Positive Rule Invariant**: If $P = \emptyset$ (for example, only negative `!` rules are listed or `patterns: []`), the result is **100% disabled** (zero tools or zero skills).

## Standard Operating Procedure (SOP)

### Step 1: Author Least-Privilege Agent YAML (`resources/configs/default/agents/*.yml`)

Expose only the minimum required `impl:deepagents` skill/tool dispatchers and domain-specific MCP/Skill namespaces:

```yaml
tools:
  patterns:
    - impl:deepagents:get_tool
    - impl:deepagents:run_tool
    - impl:deepagents:fetch_skills
    - impl:deepagents:get_skill
    - mcp:msprof-mcp:*
    - "!mcp:msprof-mcp:*_debug"
  use_catalog: false

skills:
  patterns:
    - profiler:ascend-profiler-data-validation
    - profiler:ascend-profiler-db-explorer
    - "!default:op-*"
  use_catalog: false
```

### Step 2: Verify Two-Layer Runtime Enforcement

When modifying `src/msagent/configs/`, `AgentFactory.create`, or `src/msagent/middlewares/`:
1. Confirm **Layer 1 (`AgentFactory.create`)** filters the bound tool list and caches the filtered skill registry as the primary backing store for `fetch_skills` / `get_skill`.
2. Confirm **Layer 2 (`model-call middleware`)** re-applies the exact same positive/negative pattern filter to the active tool definitions before each LLM call so DeepAgents internal default tools cannot leak into the model context.
3. Run smoke verification by launching `msagent` with the target profile and querying available MCP servers, tools, and skills to confirm excluded `!` patterns never appear.
