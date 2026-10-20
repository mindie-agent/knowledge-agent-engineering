---
category: axiom
conditions:
  component: cannbot-sentry/packages/cpx
  repository: cann/cannbot-sentry
  revision: c894c239444bce00cea23679de074df1fc4ed51d
  workload:
  - Transparent localhost LLM wire proxy capture, credential scrubbing, and zero-residue
    CLI process wrapping
created_at: '2026-10-21'
domain_path: audit/sentry-sift-insight/credential-chokepoint-cpx/wire-redaction-invariants
estimate_tokens: 580
evidence_strength: software_contract
id: unit-cpx-single-chokepoint-wire-redaction-and-zero-residue-axioms
inherits_from:
- unit-multi-agent-npu-lease-and-container-isolation-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/cpx/ARCHITECTURE.md#L1-L240
  promotion_status: verified
  source_locator: cann/cannbot-sentry@c894c239444bce00cea23679de074df1fc4ed51d:packages/cpx/ARCHITECTURE.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: 'Defines the four architectural invariants of the cpx wire proxy: single
  write chokepoint (dispatchEmit), dual key+value credential masking, flush-before-res.end
  stream ordering, and zero-residue child env injection.'
tier: L1
title: cpx Single-Chokepoint Wire Redaction (dispatchEmit), Flush-Before-End, and
  Zero-Residue Injection Axioms
triggers:
  error_signatures:
  - dispatchEmit single chokepoint
  - OPENCODE_CONFIG_CONTENT
  - scrubString
  keywords:
  - single write chokepoint
  - first4...last4 masking
  - flush before res.end
  - zero-residue injection
updated_at: '2026-10-21'
---
# cpx Single-Chokepoint Wire Redaction (dispatchEmit), Flush-Before-End, and Zero-Residue Injection Axioms

## Core Invariant

A transparent localhost wire proxy (`cpx`) that intercepts agent LLM traffic (`opencode`, `claude`, `codex`) handles live API keys, bearer tokens, and internal tool payloads. To guarantee that no plaintext secret ever touches the filesystem and no broken proxy URL survives after process exit, `cpx` enforces **four non-negotiable architectural axioms** (`packages/cpx/ARCHITECTURE.md`).

## Axiom 1: Single Write Chokepoint (`dispatchEmit()`)

1. **Zero Direct Disk Writes Outside `dispatchEmit()`**: Every protocol parser (`Anthropic`, `OpenAI ChatCompletions`, `OpenAI Responses`) and lifecycle hook must route 100% of captured events through a single synchronous chokepoint function `dispatchEmit(record)`.
2. **Mandatory Pre-Serialization Scrubbing**: `dispatchEmit()` executes recursive credential scrubbing (`scrubObject` + `scrubString`) **before** `JSON.stringify` and `fs.writeSync`. Because no other module holds a file descriptor to the output `.jsonl` stream, a new protocol adapter cannot accidentally bypass redaction.

## Axiom 2: Dual-Layer Credential Scrubbing (`first4...last4`)

Redaction inside `dispatchEmit()` applies two complementary passes to every header, JSON field, and string body:
1. **Key-Name Structural Pass**: Any object property or HTTP header whose normalized key matches sensitive credential names (`authorization`, `x-api-key`, `api-key`, `api_key`, `cookie`, `set-cookie`, `proxy-authorization`, `secret`, `password`, `token`, `access_token`, `refresh_token`) has its value masked to `first4...last4` (preserving only the first 4 and last 4 characters for key-rotation debugging, or `***` if shorter than 12 characters).
2. **Value-Shape Regex Pass (`scrubString`)**: Even inside unstructured tool stdout or prompt strings, regex scanners match known token prefixes (`sk-ant-...`, `sk-proj-...`, `sk-...`, `Bearer ...`, `ghp_...`, `glpat-...`, `AKIA...`) and mask them to `first4...last4`.

## Axiom 3: Flush-Before-`res.end()` Anti-Race Invariant

When proxying Server-Sent Events (`SSE`) or chunked responses back to the agent CLI:
- The proxy **must complete assembling the turn record and flush `dispatchEmit()` to disk before calling `res.end()`** on the downstream localhost HTTP socket.
- If `res.end()` is called first, the child agent CLI (`claude -p` or `codex exec`) immediately receives the final token, finishes its process, and triggers the parent's `child.on('exit')` handler—which closes and compresses the log file while the proxy's asynchronous `res.on('close')` handler is still mid-write, truncating the final turn!

## Axiom 4: Zero-Residue Ephemeral Child Configuration

`cpx` must **never** mutate persistent user configuration files (`~/.claude.json`, `~/.config/opencode/config.json`, or `~/.codex/config.toml`) to point `base_url` at the localhost proxy—if the process is killed with `SIGKILL`, mutated user dotfiles permanently break the user's CLI. Instead, inject the localhost proxy endpoint strictly via **process-scoped ephemeral environment variables and CLI flags** (`ANTHROPIC_BASE_URL`, `OPENAI_BASE_URL`, `OPENCODE_CONFIG_CONTENT`, `claude --settings`, `codex -c`) that vanish automatically when the child process exits.
