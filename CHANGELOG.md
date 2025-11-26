# Changelog

All notable changes to this project will be documented in this file.

## [v0.1.1-spec] - 2025-11-25
### Added
- Documented client-side relay QoS reputation (latency, throughput, uptime, integrity/consistency probes, active/backup/blacklist states, churn) in whitepaper §5.3/§8 and RFC-0004.
- Expanded threat model to cover malicious relays and the mitigations (multi-path routing, consistency probes, QoS scoring).
- README now points to current tag and relay QoS reputation coverage.

### Breaking
- Relay authentication now requires an explicit `AUTH_OK` acknowledgment per RFC-0004; legacy “silent success” or generic `OK` echoes are no longer considered valid auth success.

## [v0.1.0-spec] - 2025-11-25
### Added
- Introduced bounded `clock_skew_seconds` with bootstrap overrides.
- Clarified key bootstrapping, recovery-key usage, and signature validity windows.
- Added correction/retraction semantics and ATTESTATION_RETRACTION.
- Added stake proof registry and explicit trust budget refund rules.
- Tightened AUTH challenge validity bounds.

## [v0.0.1-spec] - 2025-11-25
### Added
- Initial draft of whitepaper and RFCs (0000–0005).
