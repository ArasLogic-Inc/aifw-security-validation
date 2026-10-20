# Session 7 — Installer Lifecycle

**Date**: October 18, 2026
**Branch**: `dev/v0.13`
**Commit**: `b337ef0`
**Result**: 12/12 PASS

## Purpose

Validate the full lifecycle of the AIFW installer on a managed-Linux host: install, configure, operate, update policy, and cleanly uninstall — proving that the installer leaves a production-ready deployment and a clean removal.

## Bug Found and Fixed

- **Gateway ExecStart missing `--data` flag**: Under systemd's `ProtectSystem=strict`, the gateway could not write to its state directory without the explicit `--data` flag pointing to a writable path. Fixed in the systemd unit template.

## Test Results

| # | Test | What It Proves | Result |
|---|------|----------------|--------|
| 1 | Install | Installer completes without error | PASS |
| 2 | Permissions | File ownership and modes match spec | PASS |
| 3 | Systemd | Service is active and running | PASS |
| 4 | mTLS | TLS handshake with client cert succeeds | PASS |
| 5 | Restart | Service survives stop/start cycle | PASS |
| 6 | Policy reload v1→v2 | Hot policy update, version confirmed via status | PASS |
| 7 | Anti-rollback | v2→v1 downgrade rejected, v2 still active | PASS |
| 8 | Idempotent re-install | Second install over existing completes cleanly | PASS |
| 9 | Clean stop | Service stops, port released | PASS |
| 10 | Journal verify | Audit chain intact after all operations | PASS |
| 11 | Uninstall | Service removed, binary removed, config removed | PASS |
| 12 | Data preservation | State directory preserved after uninstall (opt-in deletion) | PASS |

## Commit
b337ef0 fix: add --data flag to systemd ExecStart for ProtectSystem=strict compatibility
