# AIFW Security Validation

Independent validation results for the **ArasLogic AI Firewall (AIFW)** — a cross-platform AI traffic firewall that enforces identity-based model access control, forward-only policy versioning, and tamper-evident audit logging.

## Milestone: Managed-Linux Deployment

**26 / 26 tests PASS** across four validation sessions on a single managed-Linux host.

| Session | Focus | Tests | Result | Code Changes |
|---------|-------|-------|--------|--------------|
| [Session 6](results/session-06-smoke-test.md) | End-to-end smoke test | Build → PKI → policy → gateway → audit | PASS | 3 bugs fixed |
| [Session 7](results/session-07-installer-lifecycle.md) | Installer lifecycle | 12 tests: install, systemd, mTLS, policy reload, anti-rollback, uninstall | 12/12 PASS | 1 bug fixed |
| [Session 8](results/session-08-dual-identity.md) | Dual-identity separation | 11 tests: operator vs agent identity, model auth, journal integrity | 11/11 PASS | Validation only |
| [Session 9](results/session-09-bypass-resistance.md) | UID-scoped bypass resistance | 26 cumulative tests: nftables enforcement, packet counter evidence, restart survival | 26/26 PASS | Validation only |

Additionally, the [gate infrastructure](results/gate-infrastructure.md) validation covers the production-grade feature gating system (kill switches, environment lock, reboot survival) deployed on a staging server.

## What Was Proved

- An enrolled, unprivileged AI agent **cannot bypass the firewall** to reach its protected model provider directly (kernel-level packet rejection, not application-level filtering)
- Enforcement is **UID-scoped** — only the agent's UID is blocked; other users and the gateway are unaffected
- **Forward-only policy versioning** — the gateway rejects any policy with a lower version number
- **Operator / agent identity separation** — distinct X.509 workload identities with non-overlapping grants
- **Tamper-evident audit journal** — HMAC-SHA256 integrity chain, metadata-only (no prompt/response content)
- **Enforcement survives process restart** — verified after systemctl restart cycles
- **Feature gating survives reboot** — gate values, environment lock, and kill switches persist across full instance reboot

## What Was Not Tested

See [THREAT_MODEL.md](threat-model/THREAT_MODEL.md) for the full scope and explicit exclusions:

- Cross-host enforcement (agent and gateway on different machines)
- Windows and macOS enforcement mechanisms
- Load testing / performance characterization
- Remote audit anchoring (S3 Object Lock)
- Root compromise scenarios

## Repository Structure
├── README.md # This file
├── METHODOLOGY.md # Test approach, environment, limitations
├── results/
│ ├── session-06-smoke-test.md
│ ├── session-07-installer-lifecycle.md
│ ├── session-08-dual-identity.md
│ ├── session-09-bypass-resistance.md
│ └── gate-infrastructure.md
├── threat-model/
│ └── THREAT_MODEL.md
└── LICENSE

## About ArasLogic

[ArasLogic Inc.](https://araslogic.com) builds infrastructure for trustworthy AI systems. AIFW is part of the **Sentioora platform** — a reasoning engine with deterministic verification, now extended with AI traffic governance.

See also: [Sentioora Reasoning Benchmark](https://github.com/ArasLogic-Inc/sentioora-reasoning-benchmark)

## License

MIT — see [LICENSE](LICENSE).
