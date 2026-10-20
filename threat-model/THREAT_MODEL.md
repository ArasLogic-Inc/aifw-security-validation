# AIFW Threat Model

## What AIFW Defends Against

AIFW is designed to address threats that emerge when AI agents operate autonomously or semi-autonomously within an organization's infrastructure.

### Threat 1 — Unauthorized Model Access

**Scenario**: An AI agent, tool, or automation script connects directly to a model provider (OpenAI, Anthropic, etc.) using credentials it has access to, bypassing organizational policy on which models may be used, what data may be sent, and whether the interaction is logged.

**AIFW defense**: The agent's network access to the model provider is blocked at the kernel level (UID-scoped nftables on Linux). The only path to the provider is through the AIFW gateway, which applies policy before forwarding. The agent cannot bypass this without root access.

**Validated**: Session 9, tests 13–15, with packet counter evidence.

### Threat 2 — Policy Evasion via Rollback

**Scenario**: An attacker with file-system access downgrades the AIFW policy to a weaker version that permits traffic the current policy blocks.

**AIFW defense**: The gateway enforces forward-only policy versioning. A policy with a lower version number than the currently loaded one is rejected. The gateway continues operating under the current policy.

**Validated**: Session 7, test 7 (anti-rollback v2 → v1 rejected).

### Threat 3 — Identity Confusion

**Scenario**: An AI agent presents itself as an operator (or vice versa) to gain access to capabilities it should not have — for example, an agent accessing admin endpoints, or an operator proxying model requests that should be audited as agent traffic.

**AIFW defense**: Operators and agents are distinct X.509 workload identities with non-overlapping grants. The gateway enforces separation: operator certificates can access admin endpoints but cannot proxy model requests. Agent certificates can proxy granted models but cannot access admin endpoints.

**Validated**: Session 8, tests 1–5.

### Threat 4 — Audit Tampering

**Scenario**: After an incident, an attacker modifies or deletes audit records to conceal what happened — which agent sent what to which model, and what policy decision was made.

**AIFW defense**: The audit journal is an append-only chain with HMAC-SHA256 integrity. Each record's hash includes the previous record's hash, creating a tamper-evident chain. `aifw audit-verify` detects any modification, insertion, deletion, or reordering.

**Validated**: Sessions 6–9, journal verification after every test sequence.

### Threat 5 — Content Leakage via Audit

**Scenario**: The audit system itself becomes a data exfiltration path — prompt and response content stored in the journal is accessed by personnel or systems that should not see it.

**AIFW defense**: The journal records metadata only: identity, model, timing, policy decision, verdict. Prompt and response content is never written to the journal. When AIFW integrates with the Sentioora platform, only keyed HMAC fingerprints cross the boundary — not content, not even raw SHA-256 hashes (which are dictionary-matchable for short answers).

**Validated**: Sessions 8–9, tests 8 and 21 (every journal record inspected for content fields).

## What AIFW Does Not Defend Against

Honesty about limitations is part of a credible threat model.

### Not in scope (current milestone)

- **Root compromise**: An attacker with root access on the host can modify nftables rules, replace the gateway binary, or read the journal. AIFW assumes the host OS is not compromised.
- **Supply chain attacks on the gateway binary**: AIFW does not currently implement binary signing or secure boot verification.
- **Side-channel attacks**: Timing analysis of gateway responses, traffic analysis of encrypted connections, or memory inspection of the gateway process.
- **Denial of service against the gateway**: A flood of requests to the gateway port is not mitigated by AIFW itself (network-level rate limiting is an infrastructure concern).
- **Cross-host bypass**: If the agent and gateway are on different machines, network-level enforcement depends on infrastructure firewall rules, not UID-scoped nftables. Cross-host enforcement is designed but not yet validated.
- **Windows and macOS enforcement**: The UID-scoped nftables mechanism is Linux-specific. Platform-appropriate enforcement mechanisms for Windows and macOS are planned but not built.

### Architectural boundaries

- AIFW is strictly defensive. It monitors, controls, and audits. It does not modify, inject into, or weaponize AI traffic.
- AIFW's integration with the Sentioora reasoning platform is metadata-only. Prompt and response content does not cross the boundary in either direction.
- AIFW does not make content-safety or toxicity judgments. It enforces identity, model authorization, and organizational policy. Content classification, if needed, is the responsibility of the model provider or a separate content-safety layer.
