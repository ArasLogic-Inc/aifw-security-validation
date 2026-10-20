# Methodology

## Test Environment

All validation was performed on a single EC2 instance running Ubuntu with Go 1.24+ and Python 3.10.12. The test environment is isolated from production infrastructure.

- **Operating system**: Ubuntu Linux (kernel 5.15+)
- **Firewall runtime**: Go-based gateway with mTLS, systemd service management
- **Identity model**: X.509 workload certificates issued by a test PKI (not shared with production)
- **Network enforcement**: nftables with UID-scoped rules
- **Audit backend**: Append-only journal with HMAC-SHA256 integrity chain

## Approach

Each session builds on the previous one. Sessions 6 and 7 made code changes (bug fixes found during testing); sessions 8 and 9 were validation-only — no code was modified.

### Session structure

1. **Pre-flight**: Confirm git HEAD, clean working tree, no leftover state from prior sessions
2. **Setup**: Build binaries, generate test PKI, install gateway, configure policy
3. **Test execution**: Each test is a named shell function with a clear pass/fail criterion
4. **Cleanup**: Uninstall gateway, remove test certificates, delete temporary files (via cleanup trap — runs even on failure)
5. **Post-flight**: Verify git state unchanged (for validation sessions) or commit the fix (for code-change sessions)

### What counts as a pass

- Every test function exits 0
- The specific assertion in each test is met (HTTP status code, journal field value, packet counter, policy version)
- No test depends on timing, ordering artifacts, or network conditions beyond localhost

### What counts as evidence

- **Packet counter evidence**: nftables counters showing rejected packets from the agent's UID to the upstream port
- **Journal inspection**: Every journal record checked for the absence of prompt/response content fields
- **Policy version confirmation**: Gateway status endpoint reports the expected policy version after reload, and rejects rollback
- **Restart survival**: Service stop → start cycle with re-verification of all enforcement properties

## Limitations

- All tests run on a single host. Cross-host enforcement (agent and gateway on different machines) was not tested.
- The upstream model provider is a controlled Go mock, not a live API. The mock returns valid OpenAI-format responses.
- nftables enforcement depends on Linux kernel support. The same approach does not apply to Windows or macOS.
- The audit journal integrity chain is verified locally. Remote anchoring (S3 Object Lock) is designed but not yet wired end-to-end.
- Load testing was not performed. Latency and throughput under concurrent agent connections are not characterized.

## Reproducibility

The test scripts are self-contained bash scripts that build the gateway from source, generate a fresh PKI, install, run all tests, and clean up. They require:

- Go 1.24+ toolchain
- Linux with nftables support
- Root access (for systemd, nftables, and user creation)
- No external network access (mock upstream runs on localhost)

The scripts are not published in this repository to protect the source code. The results documented here are the sanitized output of those scripts.
