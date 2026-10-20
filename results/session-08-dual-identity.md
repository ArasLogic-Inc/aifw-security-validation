# Session 8 — Dual-Identity Separation

**Date**: October 18, 2026
**Branch**: `dev/v0.13`
**HEAD**: `b337ef0` (validation only — no code changes)
**Result**: 11/11 PASS

## Purpose

Prove that AIFW correctly separates two workload identities — an **operator** (admin) and an **agent** (AI workload) — and enforces that each can only do what its policy grants allow. This is the foundation for multi-tenant AI governance.

## Setup

- **Operator cert**: X.509 workload certificate with principal `operator-s8`, granted admin access (status endpoint) but no model proxy
- **Agent cert**: X.509 workload certificate with principal `agent-s8`, granted proxy access to a specific model but no admin access
- **Mock upstream**: Go HTTP server on `:18444` returning OpenAI-format responses
- **AIFW gateway**: Listening on `:18443` with mTLS, policy loaded with both principals

## Test Results

| # | Test | What It Proves | Result |
|---|------|----------------|--------|
| 1 | Operator → status | Admin access works with operator cert | PASS |
| 2 | Agent → status | Admin access denied with agent cert (HTTP 403) | PASS |
| 3 | Agent → proxy (granted model) | Model proxy works for granted model | PASS |
| 4 | Agent → proxy (denied model) | Model proxy rejected for non-granted model (HTTP 403) | PASS |
| 5 | Operator → proxy | Model proxy denied for operator cert (HTTP 403) | PASS |
| 6 | Journal chain integrity | `aifw audit-verify` passes after all operations | PASS |
| 7 | Journal metadata check | Every record has identity, model, timing, verdict fields | PASS |
| 8 | Journal no-content check | No prompt or response content in any record | PASS |
| 9 | Restart | Gateway restart, both identities re-verified | PASS |
| 10 | Post-restart operator | Operator status still works | PASS |
| 11 | Post-restart agent | Agent proxy still works for granted model | PASS |

## Identity Separation Matrix

| Action | Operator | Agent |
|--------|----------|-------|
| GET /status (admin) | ✅ Allowed | ❌ Denied |
| POST /proxy (granted model) | ❌ Denied | ✅ Allowed |
| POST /proxy (other model) | ❌ Denied | ❌ Denied |

## Key Property

> **No single certificate can both administer the gateway and proxy model requests.** An operator cannot eavesdrop on agent traffic by proxying through the gateway, and an agent cannot inspect or modify gateway configuration.
