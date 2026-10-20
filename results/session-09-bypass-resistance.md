# Session 9 — UID-Scoped Bypass Resistance

**Date**: October 18, 2026
**Branch**: `dev/v0.13`
**HEAD**: `b337ef0` (validation only — no code changes)
**Result**: 26/26 PASS (cumulative with sessions 8 + 9)

## Purpose

Prove that an enrolled AI agent, running as an unprivileged Linux user with valid upstream credentials, **cannot bypass the AIFW gateway** to reach the model provider directly. This is the core security property of the managed-Linux deployment.

## Enforcement Mechanism

- **Agent user**: `aifw-agent-s9` (UID 995), unprivileged, running in a systemd cgroup
- **Gateway user**: `aifw` (separate UID), runs the AIFW gateway process
- **nftables rule**: UID-scoped OUTPUT rule blocking the agent's UID from connecting to the upstream port (`:18444`). The gateway's UID is not blocked.
- **Effect**: The agent can connect to the gateway (`:18443`), which proxies to upstream. The agent cannot connect to upstream directly.

## Test Results (Session 9 additions on top of Session 8)

| # | Test | What It Proves | Result |
|---|------|----------------|--------|
| 1–11 | All Session 8 tests | Identity separation, model auth, journal integrity | PASS |
| 12 | nftables rule installed | Rule present in OUTPUT chain, scoped to agent UID | PASS |
| 13 | Agent direct → upstream | Connection refused / timeout (blocked by nftables) | PASS |
| 14 | Packet counter > 0 | nftables counter confirms 1+ rejected packets from agent UID | PASS |
| 15 | Agent → gateway → upstream | Proxy succeeds, mock response returned | PASS |
| 16 | Gateway restart | systemctl restart aifw-gateway | PASS |
| 17 | Post-restart: agent direct still blocked | Direct connection still refused | PASS |
| 18 | Post-restart: agent proxy still works | Proxy succeeds after restart | PASS |
| 19 | Post-restart: packet counter incremented | New rejected packets counted | PASS |
| 20 | Journal chain valid after all tests | aifw audit-verify passes | PASS |
| 21 | Journal metadata-only confirmed | No prompt/response content in any record | PASS |
| 22 | nftables rule removal | Rule deleted, agent can now reach upstream directly | PASS |
| 23 | Connectivity restored | Agent direct → upstream succeeds (rule gone) | PASS |
| 24 | Persistent nftables unchanged | `/etc/nftables.conf` not modified by any test | PASS |
| 25 | Agent user removed | Test user `aifw-agent-s9` deleted | PASS |
| 26 | Final cleanup verified | No leftover services, certs, or rules | PASS |

## The Core Proof

The critical chain is tests 13–15:

1. **Test 13**: Agent UID tries to connect directly to upstream on `:18444`. Connection is blocked by the nftables OUTPUT rule scoped to that UID.
2. **Test 14**: The nftables packet counter for the agent's UID shows 1 or more rejected packets. This is kernel-level evidence — not application-level.
3. **Test 15**: The same agent UID connects to the AIFW gateway on `:18443`. The gateway proxies the request to upstream and returns the response. The agent gets its model access, but only through the firewall.

**An enrolled agent with valid upstream API credentials cannot use them directly. Every model request must pass through AIFW.**

## Enforcement Properties

| Property | How It Was Proved |
|----------|-------------------|
| Agent cannot bypass gateway | Packet counter evidence (kernel-level) |
| Agent can still reach models through gateway | Proxy response returned |
| Enforcement survives restart | Tests 17–19 after `systemctl restart` |
| Enforcement is reversible | Rule removal restores connectivity (test 22–23) |
| No persistent system changes | `/etc/nftables.conf` unchanged (test 24) |
| Audit trail covers everything | Journal chain verified after all operations (test 20) |
| No content leakage | Every journal record inspected (test 21) |

## Milestone Claim

> **One validated managed-Linux deployment where an enrolled, unprivileged AI agent cannot bypass the firewall or reach its protected upstream model provider directly.**

This milestone is met. The evidence is:
- Kernel-level packet rejection (not application-level filtering)
- Enforcement scoped to the agent's UID (not a broad network block)
- Enforcement survives process restart
- The audit trail is intact and content-free
- The entire test ran on a single host with a controlled mock upstream
