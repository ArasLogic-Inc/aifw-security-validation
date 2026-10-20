# Gate Infrastructure — Feature Gating & Kill Switches

**Date**: October 19–20, 2026
**Server**: Staging server (EC2, Ubuntu, Python 3.12.3)
**Result**: All checks PASS across Steps 0.1–0.4

## Purpose

Replace the legacy `/tmp` file-based feature switches with a production-grade gating system that survives reboot, prevents cross-environment contamination, and provides an instant kill switch for every feature — including AIFW shadow mode.

## Architecture

- **Gate file**: `/etc/sentioora/gates.toml` — root-owned, TOML format, monotonic generation number
- **Environment lock**: `/etc/sentioora/environment` — ties the gate file to a specific EC2 instance ID, preventing an AMI-cloned staging host from accepting production gates
- **Poll reload**: 2-second mtime check, no restart required for runtime gate changes
- **Per-process status**: Each gunicorn worker writes its acknowledged generation to `/run/sentioora/gates/<PID>.json`
- **CLI**: `sentioora-gate show` (display all worker states) and `sentioora-gate set` (atomic gate update with generation bump)
- **Permissions**: `root:ubuntu 0640` on gate files, `root:ubuntu 0750` on log directory, tmpfiles.d policy for `/run/sentioora`

## Gate Declarations

```toml
generation = 2

[runtime]
"aifw.shadow.enabled"  = false    # AIFW shadow mode kill switch
"aifw.shadow.content"  = false    # Whether bodies cross to AIFW
"api.http.enabled"     = true     # HTTP API platform
"platform.maintenance" = false    # Maintenance mode
"diag.call_timing"     = false    # Per-call timing diagnostics

[boot]
# Additional boot-time gates control safety-critical features
# that cannot be toggled at runtime
```

## Validation Results

### Step 0.1 — Dev Server (Unit Tests)

| Test | Result |
|------|--------|
| Gate loading from valid TOML | PASS |
| Unknown key rejection | PASS |
| Environment lock (instance ID mismatch) | PASS |
| Generation monotonicity (lower gen refused) | PASS |
| Snapshot pinning (boot gates frozen at startup) | PASS |
| Live runtime gate reads | PASS |
| Allow-list semantics (only declared keys accepted) | PASS |
| CLI show/set commands | PASS |
| **Total**: 16/16 | **ALL PASS** |

### Step 0.2 — Staging Deploy

| Check | Result |
|-------|--------|
| gates.toml installed with correct values | PASS |
| Environment file with staging instance ID | PASS |
| sentioora-gate show returns 4 identical worker snapshots | PASS |
| All workers at generation 1 | PASS |

### Step 0.3 — Kill Switch Wiring

| Check | Result |
|-------|--------|
| api.http.enabled controls readiness endpoint | PASS |
| platform.maintenance returns 503 | PASS |
| Boot gates frozen at startup | PASS |
| CLI set refuses boot gate changes at runtime | PASS |

### Step 0.4 — Configuration Durability

| Test | What It Proves | Result |
|------|----------------|--------|
| (a) Hot reload | sentioora-gate set updates gen, all 4 workers ack within poll cycle | PASS |
| (b) Restart survival | Gate values persist across systemctl restart | PASS |
| (c) File removal | Missing gates.toml leads to defaults plus warning in log, service stays up | PASS |
| (d) Bad generation | Lower generation number rejected, current state preserved | PASS |
| (e) Instance lock | Gate file from wrong instance ID rejected | PASS |
| (f) Reboot survival | Full instance reboot, service auto-starts, gates intact, tmpfiles.d recreates status directory | PASS |

### Post-Reboot Verification

| Check | Result |
|-------|--------|
| Boot ID changed (new boot confirmed) | PASS |
| Service auto-started with 4 workers | PASS |
| /run/sentioora/gates recreated by tmpfiles.d | PASS |
| gates.toml gen 2 preserved | PASS |
| 4 status files registered at gen 2 | PASS |
| Readiness endpoint returns HTTP 200 | PASS |
| Boot-gate immutability (CLI refused runtime change) | PASS |
| Zero warnings in journal this boot | PASS |
