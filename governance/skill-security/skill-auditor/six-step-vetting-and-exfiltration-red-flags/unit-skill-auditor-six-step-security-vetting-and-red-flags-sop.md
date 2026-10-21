---
schema: mindie-graded-unit/1
id: unit-skill-auditor-six-step-security-vetting-and-red-flags-sop
revision: 1
title: "Agent Skill Security Auditor: Six-Step Pre-Deployment Vetting, Permission Toxicity Matrix, and Exfiltration Red Flags SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: governance/skill-security/skill-auditor/six-step-vetting-and-exfiltration-red-flags
inherits_from:
  - unit-okf-v02-bundle-profiles-provenance-and-trust-tier-axioms
  - unit-ascendc-vv-cv-operator-and-skill-governance-check-sop
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: governance
triggers:
  error_signatures:
    - "AGENT AUDIT REPORT"
    - "VERDICT: BLOCK"
    - "VERDICT: DANGEROUS"
  keywords:
    - "skill-auditor"
    - "typosquatting"
    - "prompt-injection"
    - "exfiltration-detection"
    - "permission-analysis"
provenance:
  source_type: source_contract
  source_locator: Ascend/agent-skills@942f0138aa9c78467507babb09aa9c64c5f534dc:skills/skill-auditor/SKILL.md
  evidence_refs:
    - https://github.com/Ascend/agent-skills/blob/942f0138aa9c78467507babb09aa9c64c5f534dc/skills/skill-auditor/SKILL.md#L1-L260
  supersedes: []
---

## Falsifiable Claim

Deploying third-party or updated AI agent Skills (`SKILL.md`, prompt templates, and helper scripts) without static security auditing exposes the host workspace to typosquatted packages, hidden prompt injections, and credential exfiltration. Executing the six-step static `skill-auditor` protocol (**never executing the target agent/skill during audit**) detects toxic permission combinations (`network + fileRead`, `network + shell`, `shell + fileWrite`), normalizes obfuscated Unicode/Base64/HTML-comment payloads before scanning, and classifies skills into `SAFE`, `SUSPICIOUS`, `DANGEROUS`, or `BLOCK` with an explicit sandbox deployment plan.

## Six-Step Static Security Audit Protocol

### Step 1: Metadata & Typosquat Verification
- Verify `SKILL.md` frontmatter (`name`, semver `version`, `description`, identifiable `author`/`source`).
- Screen `name` and referenced packages against six typosquatting techniques: missing char (`gihub-push`), extra char (`lodashs`), char swap (`code-reveiw`), homoglyphs (`babe1` L$\rightarrow$1), scope confusion (`@tyeps/node`), and hyphen/underscore substitution (`react_dom`).

### Step 2: Permission & Over-Privilege Toxicity Matrix
Compare requested capabilities (`fileRead`, `fileWrite`, `network`, `shell`) against the skill's declared purpose (e.g., a read-only code reviewer must never request `network + shell`) and flag toxic combinations immediately:

| Capability Combination | Severity | Attack Vector |
| :--- | :--- | :--- |
| **`network` + `fileRead`** | `CRITICAL` | Reads arbitrary workspace/credential files and exfiltrates them over HTTP/DNS |
| **`network` + `shell`** | `CRITICAL` | Remote command execution and reverse-shell / command-output exfiltration |
| **`shell` + `fileWrite`** | `HIGH` | Modifies system startup files (`.bashrc`, `crontab`) or persists local backdoors |
| **`fileWrite` + credential paths** | `CRITICAL` | Direct tampering with SSH keys, `.env`, or cloud/NPU cluster credentials |

### Step 3: Supply-Chain Dependency Audit
For any `pip install`, `npm install`, `go get`, or `apt install` in the skill:
- Reject packages with CVSS $\ge 9.0$ (`Critical`), unpatched CVSS $7.0\text{–}8.9$ (`High`), `preinstall`/`postinstall` hooks, obfuscated source, or unexpected `subprocess`/`child_process`/`net`/`dns` imports.

### Step 4: Normalization-First Prompt Injection Scan
1. **Pre-Scan Normalization**: Decode Base64 strings, expand Unicode escapes, strip zero-width characters (`U+200B`, `U+200C`, `U+200D`, `U+FEFF`), and flatten HTML/Markdown comments (`<!-- ... -->`).
2. **Pattern Scan**:
   - **Critical (Block)**: `"Ignore previous instructions"`, `"System prompt override"`, fake role tags (`[SYSTEM]`, `[ADMIN]`, `[ROOT]`), `"Bypass safety checks"`.
   - **High (Flag)**: `"End of system prompt"`, hidden comment directives, zero-width character steganography.

### Step 5 & Step 6: Network Exfiltration & Content Red Flags
- **Exfiltration Signatures**: Block raw external IP endpoints, DNS tunneling (`dns.resolve(${data}.domain)`), environment-variable query injection (`fetch(url?key=${process.env.API_KEY})`), custom Base64 HTTP headers, and slow-drip multi-request chunking.
- **Content Red Flags**: Block references to `.ssh`, `.aws`, `.env`, reverse shells (`bash -i`, `nc`, `powershell -e`), and instructions to disable sandboxing.

## Verdict & Safe-Deployment Gate

Emit a structured `AGENT AUDIT REPORT` with one of four deterministic verdicts:
- `SAFE` $\rightarrow$ Normal production deployment.
- `SUSPICIOUS` $\rightarrow$ Manual review + mandatory staging sandbox with restricted paths/endpoints.
- `DANGEROUS` $\rightarrow$ Blocked from deployment.
- `BLOCK` $\rightarrow$ Quarantine immediately and report to security team.
- **Re-Vetting Invariant**: `v1.0 SAFE` never implies `v1.1 SAFE`; every version update that modifies prompts, scripts, or permissions requires a fresh six-step audit.
