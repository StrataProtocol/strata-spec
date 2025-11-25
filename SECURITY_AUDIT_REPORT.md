# Strata Protocol Security Audit Report (v2)

**Audit Date:** 2025-11-25
**Auditor:** Security Review
**Spec Version:** Draft (post-hardening update)

---

## Executive Summary

The Strata Protocol specifications have undergone significant improvements since the initial audit. **The spec team has successfully addressed all 22 previously identified issues**, including all critical and high-risk vulnerabilities. The updated specifications demonstrate a mature understanding of the security requirements for a decentralized verification layer.

However, this follow-up audit has identified **8 new issues** (2 medium-severity, 6 low-severity) that warrant attention before production deployment. These are primarily edge cases, implementation ambiguities, and potential attack vectors that emerged from the new mechanisms introduced to fix the original issues.

### Summary of Changes

| Category | Previous Issues | Resolved | New Issues |
|----------|-----------------|----------|------------|
| Critical | 4 | 4 (100%) | 0 |
| High | 4 | 4 (100%) | 0 |
| Medium | 4 | 4 (100%) | 2 |
| Low | 10 | 10 (100%) | 6 |
| **Total** | **22** | **22** | **8** |

---

## Part 1: Previously Identified Issues - Resolution Status

### Critical Issues (All Resolved)

#### 1. Key Revocation Mechanism - **RESOLVED**
**Location:** RFC-0001 §5.4 (lines 153-177)

**Previous Issue:** No specification for key revocation.

**Resolution:** The spec now defines a comprehensive `KEY_EVENT` packet system:
- Events: `ADD`, `ROTATE`, `REVOKE`
- Validity windows: `valid_from`, `valid_until`, `revoked_at`
- Relay-observed time enforcement (not author timestamps)
- Requirement that relays index and serve KEY_EVENT chains
- Clear rules: signatures valid only within `[valid_from − clock_skew, min(valid_until, revoked_at) + clock_skew]`

**Verdict:** Excellently addressed. The relay-witnessed time requirement closes the backdating vulnerability.

---

#### 2. Genesis Event Forgery Risk - **RESOLVED**
**Location:** RFC-0002 §6 (lines 201-273)

**Previous Issue:** No hardware attestation binding or certificate chains.

**Resolution:** Genesis Events now include:
- `issuer_chain`: Certificate chain from device to manufacturer roots
- `attestation_evidence`: Structured evidence (Android Key Attestation, Apple DeviceCheck, C2PA)
- Verification requirements: MUST validate chain against hardware/model roots from bootstrap documents
- Explicit downgrade: missing/invalid chain → `UNKNOWN/SOFTWARE`

**Verdict:** Well addressed. The certificate chain model follows industry best practices (C2PA alignment).

---

#### 3. Timestamp Manipulation - **RESOLVED**
**Location:** RFC-0002 §3.3 (lines 98-106)

**Previous Issue:** Timestamps were self-reported with no verification.

**Resolution:**
- Relays MUST reject packets >5 minutes in future or >24 hours in past
- Relays MUST record and serve `received_at` (relay-observed timestamp)
- Clients MUST prefer `received_at` for quorum/age calculations
- Age gating in RFC-0005 §5.1 explicitly uses `received_at`, not author timestamps

**Verdict:** Comprehensively addressed. The dual timestamp system (author + relay-witnessed) is sound.

---

#### 4. Sybil Attack on Bootstrap - **RESOLVED**
**Location:** RFC-0005 §4.1 (lines 63-73), Whitepaper §10.1

**Previous Issue:** Bootstrap documents lacked authentication and diversity requirements.

**Resolution:** Bootstrap documents now MUST:
- Be content-addressed (`bootstrap_id = multihash(blake3-256(doc))`)
- Be signed by at least 3 distinct gatekeepers with different `diversity_tags`
- Include `issued_at` and `expires_at` for time-bounding
- Include hardware_roots and model_roots for genesis verification
- Clients MUST fail closed if signatures fall below threshold

**Verdict:** Well addressed. The diversity tag system and multi-signature threshold provide meaningful Sybil resistance at bootstrap.

---

### High-Risk Issues (All Resolved)

#### 5. Edit History Integrity - **RESOLVED**
**Location:** RFC-0002 §5 (lines 142-191)

**Previous Issue:** Edit history signatures were optional.

**Resolution:**
- For `HARDWARE_SECURE_ENCLAVE` and `AI_MODEL`: edit_history MUST form a hash chain with no gaps
- Signature is REQUIRED for hardware/AI origin to maintain green-ring status
- Missing or broken chains explicitly downgrade provenance to `UNKNOWN/YELLOW`
- Each edit entry includes `prev_edit_hash` creating a verifiable chain

**Verdict:** Appropriately addressed with clear downgrade semantics.

---

#### 6. Replay Attacks - **RESOLVED**
**Location:** RFC-0002 §3.1, §8 (lines 302-308)

**Previous Issue:** No nonce or sequence mechanism; packets could be replayed.

**Resolution:**
- `expires_at` field for absolute TTL
- `nonce` field for replay detection
- Relays MUST maintain dedup cache for at least 7 days
- Relays MUST reject packets older than backfill window (default 24h)
- Attestation packets deduplicated by `attestation_id`/`target_packet` pairs

**Verdict:** Comprehensively addressed. Multi-layer defense (nonce, dedup, TTL, age limits).

---

#### 7. Trust Edge Gaming - **RESOLVED**
**Location:** RFC-0001 §7.3-7.4 (lines 231-258)

**Previous Issue:** No explicit revocation type; wash-trading possible.

**Resolution:**
- Explicit `TRUST_REVOCATION` packet type defined
- Revocation MUST set effective strength to zero for all earlier edges
- `strength: 0` TRUST_EDGE explicitly is NOT a revocation
- Concrete trust budgets: max 20 edges with `strength > 0.5` per 24h
- Revocations don't immediately refund budget (prevents wash-trading)

**Verdict:** Well addressed with concrete parameters and anti-gaming measures.

---

#### 8. Consensus Metrics Authentication - **RESOLVED**
**Location:** RFC-0002 §7 (lines 274-301)

**Previous Issue:** Consensus metrics could be manipulated by authors.

**Resolution:**
- In-packet `consensus_metrics` field is RESERVED
- Authors MUST NOT populate it; clients MUST ignore unsigned values
- Metrics MUST be published as standalone `CONSENSUS_METRIC` packets
- Must be signed by `source_id` (relay/aggregator)
- Clients treat as advisory evidence tied to source reputation

**Verdict:** Clean solution that makes metrics attributable and verifiable.

---

### Medium-Risk Issues (All Resolved)

#### 9. DID Resolution - **RESOLVED**
**Location:** RFC-0001 §3.3 (lines 97-110)

**Resolution:** Deterministic resolution order defined:
1. Local cache with freshness window
2. Bootstrap documents and relay manifests for `did_doc_hash` pointers
3. Fetch content-addressed document by hash

Plus: monotonic updates via `prev_did_doc_hash`, conflict handling (fail closed), gossip recommendations.

---

#### 10. Private Message Encryption - **RESOLVED**
**Location:** RFC-0004 §5.1 (lines 223-252)

**Resolution:** Concrete specification:
- Key agreement: X25519
- AEAD: XChaCha20-Poly1305
- KDF: HKDF-SHA256
- Forward secrecy: Double Ratchet or MLS
- Envelope format with scheme identifier and per-recipient ciphertext

---

#### 11. Rate Limiting Specification - **RESOLVED**
**Location:** RFC-0004 §6.1 (lines 276-284)

**Resolution:** Standardized error response with `retry_after`, `limit`, `remaining`, `window_seconds` fields. Scoping to authenticated identity preferred.

---

#### 12. Stake Mechanism - **RESOLVED**
**Location:** RFC-0005 §4.3 (lines 87-121)

**Resolution:** Detailed stake structure with:
- Asset specification (symbol, chain, decimals)
- Proof requirement (onchain-tx with tx_id, block_height, proof_blob)
- Slashing via `STAKE_SLASH` packets with evidence
- Rules for handling unverified stakes (down-weight to zero)

---

### Low-Risk Issues (All Resolved)

| Issue | Resolution |
|-------|------------|
| 13. RFC Reference Mismatch | RFC-0000 §3.5 now correctly references RFC-0003 and RFC-0005 |
| 14. Hash Function Inconsistency | BLAKE3-256 is now MUST with multihash; SHA-256 legacy-only |
| 15. packet_id Encoding | Multibase of multihash is MUST; hex is display-only |
| 16. Trust Edge Signature Scope | Clarified: edge_signature OPTIONAL, created_at handling defined |
| 17. No UNSUBSCRIBE | RFC-0004 §3.6 now defines UNSUBSCRIBE message |
| 18. No Pagination | RFC-0004 §3.1 now has limit, cursor, order fields |
| 19. No Corrections | RFC-0002 §3.4 defines CORRECTION mechanism |
| 20. No Relay Authentication | RFC-0004 §3.7 defines AUTH_CHALLENGE/AUTH flow |
| 21. No Media Storage Spec | RFC-0002 §5.1 defines responsibility and availability |
| 22. No Bootstrap Format | Whitepaper §10.1 has full schema with validation rules |

---

## Part 2: New Issues Identified in Updated Specifications

### Medium Severity Issues

#### NEW-1: Clock Skew Parameter Undefined
**Location:** RFC-0001 §5.4 (line 175-176), RFC-0002 §3.3

**Issue:** The spec references `clock_skew` multiple times as a tolerance parameter for key validity and timestamp checks, but:
- No concrete value is specified
- No guidance on acceptable range
- No mechanism for clients/relays to agree on skew tolerance

**Code Reference:**
```
RFC-0001:175: "A Packet signature is valid only if the relay‑observed receive time
is within [valid_from − clock_skew, min(valid_until, revoked_at) + clock_skew]"
```

**Risk:** Different implementations using different skew values will produce inconsistent signature validity decisions. Too large a skew undermines time-based security; too small causes legitimate packets to be rejected.

**Recommendation:** Define a normative `clock_skew` value (suggest: 300 seconds / 5 minutes) or specify it must be included in bootstrap documents.

---

#### NEW-2: KEY_EVENT Chain Bootstrapping Problem
**Location:** RFC-0001 §5.4 (line 173)

**Issue:** KEY_EVENT packets MUST be signed by "an existing non-revoked key (or a designated offline recovery key)." However:
- The initial key has no KEY_EVENT establishing its validity
- There's no specification for how the first key's `valid_from` is determined
- Recovery key designation mechanism is unspecified

**Risk:**
- Ambiguity about when the original key becomes valid
- No clear path for bootstrapping identity after total key loss
- Recovery key could be a backdoor if not properly specified

**Recommendation:**
1. Specify that the genesis DID Document establishes initial key validity
2. Define recovery key registration in DID Document
3. Specify recovery key constraints (e.g., must be offline, hardware-backed)

---

### Low Severity Issues

#### NEW-3: Diversity Tag Collusion
**Location:** RFC-0005 §4.1 (line 67), Whitepaper §10.1

**Issue:** Bootstrap documents require signatures from gatekeepers with "different `diversity_tags`" (region/org_type), but:
- Diversity tags are self-asserted
- No verification that tags accurately reflect diversity
- A single entity could register multiple gatekeepers with different tags

**Example Attack:**
```
Attacker registers:
- did:strata:attacker_1 with diversity_tags: ["region:eu", "org:ngo"]
- did:strata:attacker_2 with diversity_tags: ["region:na", "org:foundation"]
- did:strata:attacker_3 with diversity_tags: ["region:asia", "org:collective"]
```

**Risk:** The diversity requirement provides only weak protection against a determined adversary with resources to establish multiple legal entities.

**Recommendation:** Consider requiring diversity tags to reference verifiable external anchors (e.g., domain verification, legal entity registration proof).

---

#### NEW-4: Stake Proof Verification Ambiguity
**Location:** RFC-0005 §4.3 (lines 102-107)

**Issue:** The stake `proof` object specifies `type: "onchain-tx"` but:
- No specification for how to verify proofs for different chains
- No list of supported chains/proof formats
- `proof_blob` format is undefined ("base64merkleproof...")
- No specification for off-chain escrow verification (`chain: "offchain-escrow"`)

**Risk:** Implementations may accept invalid proofs or reject valid ones due to format mismatches.

**Recommendation:** Define a registry of supported chains with their proof verification algorithms, or specify a generic proof verification interface.

---

#### NEW-5: CORRECTION Packet Supersession Semantics
**Location:** RFC-0002 §3.4 (lines 107-114)

**Issue:** CORRECTION packets reference `target_packet` but:
- No mechanism to prevent multiple conflicting corrections
- No priority/ordering when multiple corrections exist
- "Should down-rank or blur superseded Packets" is vague on degree

**Scenario:** Author publishes corrections A and B for the same packet with different replacement text. Which is authoritative?

**Risk:** Inconsistent client behavior; potential for confusion or manipulation.

**Recommendation:** Specify that only the author can issue corrections (already implied but not explicit), and define ordering by `timestamp` with latest-wins semantics.

---

#### NEW-6: Incomplete Trust Budget Refund Specification
**Location:** RFC-0005 §5.2 (line 149)

**Issue:** "Revocations don't refund budget immediately; credit restored after same window" but:
- What happens if the window spans multiple revocations?
- Does revoking a low-strength edge affect the high-strength budget?
- No specification for what "same window" means when revocation spans window boundary

**Risk:** Edge cases could allow gaming the budget system.

**Recommendation:** Provide explicit formulas or pseudocode for budget accounting, including edge cases.

---

#### NEW-7: AUTH Challenge Expiration Race
**Location:** RFC-0004 §3.7 (lines 175-201)

**Issue:** AUTH_CHALLENGE has `expires_at` but:
- No guidance on minimum validity period
- No specification for retry behavior if challenge expires during signing
- A malicious relay could issue very short-lived challenges to deny service

**Risk:** Poor UX or potential DoS vector.

**Recommendation:** Specify minimum challenge validity (suggest: 60 seconds) and recommended client retry behavior.

---

#### NEW-8: Attestation Retraction via CORRECTION Ambiguity
**Location:** RFC-0002 §3.4 (line 112)

**Issue:** "Attestors retracting a mistaken attestation SHOULD emit a CORRECTION Packet that supersedes the earlier attestation by ID." However:
- CORRECTION is defined for content corrections, not attestation retractions
- Relationship between CORRECTION and attestation validity is unclear
- Does a corrected attestation still count toward quorum?

**Risk:** Ambiguous attestation lifecycle could be exploited to game retroactive consensus.

**Recommendation:** Either define a separate `ATTESTATION_RETRACTION` packet type or explicitly specify how CORRECTION interacts with attestation quorum calculations.

---

## Part 3: Attack Scenario Re-evaluation

### Previous Attack Scenarios - Status

#### Attack Scenario A: "Reputation Laundering" - **MITIGATED**

The updated spec addresses this through:
- Trust budgets limiting edge issuance (20 high-strength per 24h)
- Revocations don't refund budget immediately
- Age gating on new identities
- Cluster detection recommendations

**Residual Risk:** LOW - A determined attacker could still slowly launder reputation over months, but the friction is significantly higher.

---

#### Attack Scenario B: "Quorum Capture" - **MITIGATED**

The updated spec addresses this through:
- `T_min` requirement using relay-witnessed timestamps
- Cluster diversity requirement (`C_min`)
- Age gating prevents flash-mob attestor creation
- Relay deduplication prevents replay

**Residual Risk:** LOW - Long-term infiltration still possible but requires sustained investment.

---

#### Attack Scenario C: "Genesis Event Injection" - **MITIGATED**

The updated spec addresses this through:
- Mandatory `issuer_chain` for hardware claims
- Chain must trace to known manufacturer roots in bootstrap documents
- `attestation_evidence` requirement (Android Key Attestation, etc.)
- Missing chain → automatic downgrade to UNKNOWN

**Residual Risk:** MEDIUM - Relies on hardware vendors' key security. A compromised manufacturer key would be catastrophic but is outside protocol scope.

---

#### Attack Scenario D: "Trust Graph Pollution" - **MITIGATED**

The updated spec addresses this through:
- Trust budgets (20 high-strength edges per 24h)
- Age gating during grace period
- Cluster detection recommendations
- Revocations don't restore budget immediately

**Residual Risk:** LOW - Significantly harder to pollute the graph quickly.

---

### New Attack Scenario Identified

#### Attack Scenario E: "Clock Skew Arbitrage"

**Premise:** Different relays use different `clock_skew` tolerances.

**Attack Flow:**
1. Attacker identifies relays with large clock_skew tolerance (e.g., 1 hour)
2. Attacker publishes KEY_EVENT revoking their key at time T
3. Before revocation propagates, attacker uses permissive relay to publish malicious packets with timestamps near T
4. Some clients accept these packets (via permissive relays) while others reject them
5. Network has inconsistent view of which packets are valid

**Impact:** Creates uncertainty about packet validity; can be used to selectively target users of permissive relays.

**Mitigation Needed:** Standardize `clock_skew` parameter or include it in relay manifests so clients can make informed decisions.

---

## Part 4: Recommendations Summary

### Immediate (Pre-Implementation)

1. **Define `clock_skew` parameter** - Critical for consistent key validity
2. **Specify KEY_EVENT bootstrapping** - Clarify initial key validity
3. **Clarify CORRECTION semantics** - Define ordering and attestation interaction

### Short-Term

4. **Enhance diversity tag verification** - Consider external anchors
5. **Define stake proof verification interface** - Needed for implementation
6. **Complete trust budget specification** - Handle edge cases

### Long-Term

7. **AUTH challenge robustness** - Minimum validity, retry behavior
8. **Formal threat model update** - Document new mechanisms

---

## Conclusion

The Strata Protocol has made remarkable progress in addressing the security concerns identified in the initial audit. **All 22 previously identified issues have been resolved**, many with comprehensive solutions that go beyond the minimum requirements.

The protocol now has:
- Robust key lifecycle management with relay-witnessed time
- Hardware attestation binding via certificate chains
- Anti-replay mechanisms at multiple layers
- Concrete anti-Sybil parameters
- Complete encryption specification for private messages
- Comprehensive bootstrap document security

The 8 newly identified issues are primarily:
- Implementation details that need specification (clock_skew, proof formats)
- Edge cases in new mechanisms (budget refunds, correction ordering)
- Defense-in-depth improvements (diversity verification)

**None of the new issues are critical or high-severity.** The protocol is significantly more mature and implementation-ready than in the initial review.

### Security Posture Assessment

| Aspect | Previous | Current |
|--------|----------|---------|
| Key Management | Weak | Strong |
| Provenance Verification | Weak | Strong |
| Timestamp Integrity | Weak | Strong |
| Sybil Resistance | Moderate | Strong |
| Replay Protection | Weak | Strong |
| Trust Edge Security | Weak | Strong |
| Specification Completeness | Low | High |

**Overall Assessment:** The protocol is ready for implementation with minor specification clarifications. A formal security review by cryptography specialists is still recommended before production deployment, particularly for the trust/reputation algorithms and the encryption scheme.

---

*End of Security Audit Report*
