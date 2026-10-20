# Session 6 — End-to-End Smoke Test

**Date**: October 18, 2026
**Branch**: `dev/v0.13`
**Commit**: `1f0c90b`
**Result**: PASS

## Purpose

Validate the complete manual deployment path for a managed-Linux AIFW installation: build, PKI initialization, policy configuration, gateway startup, status check, and audit journal verification.

## Bugs Found and Fixed

Three bugs were discovered and fixed during this session:

1. **Admin-principal policy violation**: Policy grants for admin principals must have empty `models` and `resources` arrays. The test policy had non-empty arrays, which the gateway silently accepted but should have rejected. Fixed in the policy validation logic.

2. **PKI init directory conflict**: The `pki-init` command failed when the target directory already existed from a prior run. Fixed to handle pre-existing directories gracefully.

3. **Missing `set -e` in test scripts**: Error propagation was unreliable without strict mode. Added to all test scripts.

Additionally, documentation errors identified by an independent reviewer were corrected:

- `egress_mode` field uses string values (e.g., `"public"`), not CIDR notation
- Audit key type is HMAC-SHA256, not Ed25519
- `--state` flag points to a directory, not a file

## Test Sequence

| Step | Action | Result |
|------|--------|--------|
| Build | `go build` with race detector | Binary produced, no race warnings |
| PKI init | Generate test CA + gateway cert | Certificates created with correct SANs |
| Policy | Load v1 policy with admin + agent principals | Policy accepted, version confirmed |
| Gateway start | Launch with mTLS on test port | Listening, TLS handshake succeeds |
| Status | Admin principal queries `/status` | HTTP 200, correct policy version, uptime > 0 |
| Audit verify | `aifw audit-verify` on journal | Chain integrity confirmed, zero errors |

## Commit
1f0c90b fix: admin grant validation, pki-init idempotency, test strictness; docs corrections
