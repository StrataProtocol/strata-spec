# Changelog

All notable changes to this project will be documented in this file.

## [v0.4.1-spec] - 2025-12-02

### Added
- **Initial Confidence Boost (Newcomer Grace):** RFC-0005 §4.5 introduces a decaying trust boost for new identities to distinguish "unknown" from "distrusted." This addresses the cold-start problem where legitimate new users would otherwise start at effectively 0% trust. Reference formula: `T_boost = B_max * V_mult * decay(age)` with client-tunable parameters.
- **Identity Gate Verification Attestations:** RFC-0001 §9 defines a new attestation type (`strata.identity.verification`) that identity gates issue to record the verification level performed during StrataID provisioning (none, email, phone, government_id, biometric). Clients use these attestations to adjust Initial Confidence Boost parameters.
- **IDENTITY_GATE attestor type:** RFC-0003 §5 adds `IDENTITY_GATE` to the attestor_type enum for services that provision StrataIDs.

### Protocol Clarifications
- **Verification-adjusted grace periods:** Higher verification levels (phone, government_id, biometric) extend the grace period and boost multiplier, providing stronger benefit-of-the-doubt for more thoroughly verified identities.
- **Boost constraints:** `T_boost` alone cannot reach green threshold (`T_green_min`), ensuring real trust signals are still required for full trust status.

## [v0.4.0-spec] - 2025-11-28

### Security Fixes
- **A1: Attestation binding vulnerability closed.** All attestations (embedded and standalone) now MUST include `target_packet` in the signed payload. Embedded attestations MUST have `target_packet` matching the host `packet_id`. New §3.4 in RFC-0003 defines attestation signature scope explicitly. This prevents claim-rebinding attacks where attestations could be copied between packets.
- **T1: Packet size limits added.** RFC-0002 §3.5 defines 256 KiB recommended / 1 MiB hard limit. RFC-0004 adds `packet_too_large` error code. Prevents memory exhaustion DoS.
- **K1: Recovery key safeguards.** RFC-0001 §5.3 now includes RECOMMENDED time-delay (24h pending window) and threshold recovery for high-value identities. §8 documents recovery key compromise risks.

### Protocol Clarifications
- **A2: Multi-relay `received_at` semantics.** Clients MUST use earliest `received_at` across relays for all age-based and quorum calculations (RFC-0002 §3.3). Relays SHOULD update to earliest on replication (RFC-0007 §6.3). Network converges on consistent witness time.
- **M1: E2EE messaging marked non-normative.** RFC-0004 §5.1 now explicitly a placeholder pending RFC-00XX. Implementations MUST NOT assume interoperability based on `scheme` strings alone.
- **S1: GROUP_MEMBERSHIP authorization model.** RFC-0200 §9 now defines authority set (OWNER + ADMIN), authorization rules for non-self membership changes, and `open_membership` flag for self-join policy.
- **S2: PROFILE resolution strengthened.** Changed from SHOULD to MUST for canonical self-profile resolution by latest `received_at`.
- **T2: Relay manifest trust language.** RFC-0007 §10 clarifies clients SHOULD NOT use unsigned manifests for security-sensitive decisions.

### Editorial
- **O1:** Added `ATTESTATION_RETRACTION` to RFC-0000 §7.3 content.type registry.
- **O2:** RFC-0005 scope clarified as "Mixed" — §4.1 is normative when referenced, rest is non-normative.
- **O3:** RFC-0001 §7.4 now cross-references RFC-0005 §5.2 for complete trust budget parameters.
- **NEW:** RFC-0005 §4.3 stake advisory warning — stake MUST be treated as advisory until dedicated RFC defines proofs/slashing.

## [v0.3.0-spec] - 2025-11-27
### Added
- Introduced an explicit `domain` axis on attestations (PROVENANCE | CONTENT | SPAM_ABUSE | OTHER) with a mandatory inference table for legacy attestations.
- Reframed claim taxonomy as domain + subject, adding `FABRICATED_EVENT` under the CONTENT domain and keeping the provenance/spam subjects intact.
- Defined a `MISINFO_FLAGGED` state for content-domain claims in the Reality Tuner, with Strict/Standard/Wild handling and optional UX labeling guidance.

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
