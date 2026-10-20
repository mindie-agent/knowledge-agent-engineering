---
category: sop
conditions:
  component: cannbot-sentry/packages/cpx
  protocols:
  - Anthropic Messages
  - OpenAI ChatCompletions
  - OpenAI Responses
  repository: cann/cannbot-sentry
  revision: c894c239444bce00cea23679de074df1fc4ed51d
created_at: '2026-10-21'
domain_path: audit/sentry-sift-insight/credential-chokepoint-cpx/proxy-capture-and-7z-lifecycle
estimate_tokens: 590
evidence_strength: software_contract
id: unit-cpx-multi-protocol-sse-capture-and-7z-archive-sop
inherits_from:
- unit-cpx-single-chokepoint-wire-redaction-and-zero-residue-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/cpx/README.md#L1-L180
  - https://gitcode.com/cann/cannbot-sentry/blob/c894c239444bce00cea23679de074df1fc4ed51d/packages/cpx/ARCHITECTURE.md#L80-L240
  promotion_status: verified
  source_locator: cann/cannbot-sentry@c894c239444bce00cea23679de074df1fc4ed51d:packages/cpx/README.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: 'Operates cpx across Anthropic/OpenAI/Responses wire protocols: reassembling
  SSE delta streams, bridging parent-child subagent toolUseId swimlanes, enforcing
  NO_PROXY on localhost, and compressing to .jsonl.7z.'
tier: L2
title: cpx Multi-Protocol SSE Reassembly, Subagent Swimlane Bridging, NO_PROXY Bypass,
  and .jsonl.7z Archive SOP
triggers:
  error_signatures:
  - .jsonl.7z
  - ProxyAgent
  - content_block_delta
  keywords:
  - cpx SSE reassembly
  - subagent toolUseId bridging
  - NO_PROXY localhost hop
  - LZMA2 7z archive
updated_at: '2026-10-21'
---
# cpx Multi-Protocol SSE Reassembly, Subagent Swimlane Bridging, NO_PROXY Bypass, and .jsonl.7z Archive SOP

## Core Principle

When wrapping heterogeneous coding agents (`opencode`, `cannbot`, `claude`, `codex`) behind `cpx`, the proxy must decouple **wire protocol parsing** from **agent framework session branching**: a single agent session may speak Anthropic `/v1/messages`, OpenAI `/v1/chat/completions`, or OpenAI `/v1/responses`, spawn multiple concurrent subagents over the same localhost port, and run behind corporate HTTP proxies.

## 1. Protocol-to-Emitter Separation & SSE Stream Reassembly

Never dump raw SSE `data:` lines directly into the session log—thousands of single-token `content_block_delta` or `response.output_text.delta` chunks bloat logs by $20\times$ and break offline analyzers:
1. **Anthropic Messages (`/v1/messages`)**: Reassemble `message_start`, `content_block_start` (`text`, `thinking`, `tool_use`), `content_block_delta` (`text_delta`, `thinking_delta`, `input_json_delta`), and `message_delta` (`usage`) into a single consolidated assistant turn record with complete `thinking`, `text`, `tool_calls`, and token `usage`.
2. **OpenAI ChatCompletions (`/v1/chat/completions`)**: Merge streamed `delta.content`, `delta.reasoning_content`, and indexed `delta.tool_calls[i].function.arguments` fragments into a unified turn object.
3. **OpenAI Responses (`/v1/responses`)**: Reassemble `response.output_item.added`, `response.function_call_arguments.delta`, and `response.completed` events into the canonical turn schema.

## 2. Subagent `toolUseId` Parent-Child Swimlane Bridging

When a parent agent spawns a subagent (`Task` / `dispatch_agent` / `spawn_agent`), both parent and subagent send HTTP requests to the same localhost `cpx` port without an explicit thread ID header:
- **Fingerprint Matching**: Match the parent's pending `tool_use` call (`name in {"Task", "dispatch_agent", "spawn_agent"}`, `input.prompt`) against the incoming subagent request's initial user message prompt.
- Stamp every turn belonging to the subagent conversation branch with `parentToolUseId = <tool_use.id>` and `subagentId`, enabling `cannbot-insight` and `sift` to render nested subagent swimlanes without mixing subagent turns into the main agent's context window growth curve.

## 3. Corporate Proxy Two-Hop Routing (`NO_PROXY` + Upstream `ProxyAgent`)

In enterprise environments where `HTTPS_PROXY` / `HTTP_PROXY` is set:
1. **Downstream Hop (Agent CLI $\to$ Localhost `cpx`)**: Inject `NO_PROXY="127.0.0.1,localhost"` (and `no_proxy`) into the child agent's environment so the agent CLI does not route `http://127.0.0.1:<ephemeral_port>` through the external corporate proxy.
2. **Upstream Hop (Localhost `cpx` $\to$ LLM Provider)**: `cpx` reads the original `HTTPS_PROXY` from its own parent environment and attaches an `undici.ProxyAgent` for outbound TLS connections to the upstream LLM gateway.

## 4. Deterministic `.jsonl.7z` At-Rest Compression Lifecycle

- During active execution, stream scrubbed records to `<session>.jsonl`.
- On child process exit (after all in-flight HTTP streams flush per Axiom 3), compress `<session>.jsonl` into `<session>.jsonl.7z` using `7z` (`LZMA2` solid compression, achieving $\sim 10\times\text{--}15\times$ reduction on repetitive context windows) and remove the uncompressed `.jsonl` only after verifying `7z` exit code `0`.
- On `--resume`, transparently extract `<session>.jsonl.7z` back to `.jsonl`, append new turns, and re-compress on exit.
