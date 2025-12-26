<!-- rfcs/rfc-0001-identity-and-strataid.md -->

# RFC-0001: Identity and StrataID

- **Status:** Draft
- **Author(s):** Strata Core Team
- **Created:** 2025-11-25
- **Updated:** 2025-12-25
- **Scope:** Normative protocol (identity, DIDs, trust edges)

---

## 1. Summary

This RFC defines the **identity model** of the Strata Protocol:

- StrataIDs as decentralized identifiers (`did:strata`),
- Key management expectations,
- Support for multiple personas per human,
- Trust edges as the basis for Web‑of‑Trust.

---

## 2. Motivation

Strata aims to:

- Decouple identity from any specific platform or service,
- Allow reputation to travel with keys instead of accounts,
- Provide a foundation for Sybil‑resistant trust and filtering.

A formal identity model is required so that all Strata‑compatible components agree on how to:

- Represent identities,
- Verify signatures,
- Attach trust information.

---

## 3. StrataID Format

### 3.1 DID Method

Strata defines the DID method:

```text
did:strata:<id-string>
```
Where:

- `did` – standard DID prefix.
- `strata` – Strata method name.
- `<id-string>` – method‑specific identifier, typically derived from a public key.

The exact encoding of `<id-string>`:
- **MUST** use a multibase/multicodec scheme.
- **SHOULD** be compatible with existing DID and multiformat tooling.

Example:
```text
did:strata:z6Mkf5k7hEwQ1...
```

### 3.2 DID Document

A Strata DID Document MUST include:

- At least one verification method.

```json
{
  "id": "did:strata:z6MkfQ...",
  "verification_method": [
    {
      "id": "did:strata:z6MkfQ#keys-1",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:strata:z6MkfQ",
      "public_key_hex": "0x...",
      "public_key_multibase": "z6MkfQ...",
      "valid_from": 1715421200
    }
  ],
  "authentication": ["did:strata:z6MkfQ#keys-1"],
  "assertion": ["did:strata:z6MkfQ#keys-1"],
  "created_at": 1715421200,
  "prev_did_doc_hash": null
}
```

> **Wire Format Note:** Per RFC-0000 §4.3, Strata uses `snake_case` field names (e.g., `verification_method`, `public_key_multibase`) rather than W3C DID Core's `camelCase`. The `@context` field is omitted in wire format as Strata DID Documents are self-describing.

- It **MAY** include:
  - Additional keys (for rotation or multi‑device),
  - `recovery_policy` for threshold recovery requirements (see §5.3.1),
  - Service endpoints (e.g., preferred relays),
  - Metadata (e.g., human‑readable name).

Strata does not mandate a specific DID Document registry; DID Documents can be:

- Stored via content addressing and resolved through relays,
- Anchored in external systems (blockchains, key servers),
- Distributed through bootstrap documents.

### 3.3 DID Resolution & Distribution

Clients **MUST** follow a deterministic resolution order to avoid split-brain identity views:

1. Check locally cached DID Document referenced by its multihash (`did_doc_hash`) seen within a freshness window.
2. Query bootstrap documents (whitepaper §10.1) and relay manifests for `did_doc_hash` pointers for the target DID.
3. Fetch the content-addressed DID Document by `did_doc_hash` from relays or other stores.

Rules:
- DID Documents **MUST** be content-addressed; `did_doc_hash = multihash(blake3-256(canonical_did_doc))` encoded as **base58btc multibase**. Other encodings (hex) MUST be rejected.
- Updates **MUST** be monotonic and linked via `prev_did_doc_hash` to allow replay/rollback protection.
- If multiple heads exist for the same DID (conflict), clients **MUST** fail closed and surface the conflict instead of guessing.
- Relay operators **SHOULD** gossip `did_doc_hash` updates alongside identity-scoped packets to speed propagation.

#### 3.4 Reference Resolver (Non-Normative)

A shared resolver library is RECOMMENDED so relays/clients do not embed bespoke key registries. The reference implementation is `strata-identity` (Layer 0), which:

- Resolves `did:strata:...` → current Ed25519 verification key using a deterministic chain of sources (bootstrap docs, static maps, cached DID Documents),
- Verifies `AUTH` and packet signatures using canonical JSON and BLAKE3-based `packet_id`,
- Exposes both Go and JS/TS APIs so relays and clients can share identical resolution logic.

Implementations MAY use alternative resolvers provided they honor the resolution order and validation rules in this RFC and RFC-0002.

### 4 Cryptographic Primitives

Implementations **MUST** support:

- Identity keys: Ed25519 keypairs.
- Signatures: Ed25519 signatures over canonicalized JSON structures.

Future versions **MAY** introduce additional algorithms; these **MUST** be indicated via appropriate multicodec/multibase tags.

### 5. Key Management
#### 5.1 Generation

Clients **MUST**:

- Generate keys locally on the user’s device.
- Prefer secure hardware storage where available:
  - Secure Enclave (iOS/macOS),
  - StrongBox / hardware‑backed keystore (Android),
  - TPM or similar on desktop.

#### 5.2 Storage

- Keys **SHOULD** be stored using platform‑native secure storage APIs.

- For web PWAs:
  - WebCrypto + WebAuthn/Passkeys **SHOULD** be used where possible.
  - Fallbacks may use IndexedDB or other storage; this **MUST** be clearly indicated to the user in a user-friendly way that even grandma can understand.

#### 5.3 Recovery & Rotation

The protocol supports:

- **Key rotation:**
  - DID Documents can list new verification methods and deprecate old ones.
  - Packets include the author’s DID; signature verification uses the current or historical keys.
- **Recovery keys:**
  - DID Documents **MAY** declare dedicated recovery verification methods (e.g., `usage: "recovery"`).
  - Recovery keys **MUST** be offline or hardware-backed and **MUST NOT** sign application Packets; their only allowed use is signing `KEY_EVENT` and `RECOVERY_APPROVAL` packets when all online keys are lost or revoked.
  - Clients/relays **MUST** reject application Packets signed by recovery keys and **MUST** accept recovery-signed `KEY_EVENT` or `RECOVERY_APPROVAL` only if the recovery key is declared in the DID Document and not revoked.

#### 5.3.1 Threshold Recovery (Normative)

If a DID Document includes a `recovery_policy` object, clients and relays **MUST** enforce threshold recovery rules for recovery-signed `KEY_EVENT` packets.

```jsonc
"recovery_policy": {
  "threshold": 2,
  "key_ids": [
    "did:strata:alice#recovery-1",
    "did:strata:alice#recovery-2",
    "did:strata:alice#recovery-3"
  ]
}
```

Rules:
- `threshold` **MUST** be an integer >= 1.
- If `key_ids` is omitted or empty, all verification methods with `usage: "recovery"` are eligible.
- `threshold` **MUST** be <= the number of eligible recovery keys; otherwise clients **MUST** fail closed on recovery-signed `KEY_EVENT`s.
- A recovery-signed `KEY_EVENT` **MUST** be held pending until approvals from at least `threshold` distinct eligible recovery keys are observed. The signing recovery key counts as one approval.

Recovery approvals are expressed as a `RECOVERY_APPROVAL` Packet:

```jsonc
{
  "type": "RECOVERY_APPROVAL",
  "target_event": "0x...",  // packet_id of the KEY_EVENT being approved
  "approved_at": 1715421200,
  "note": "optional"
}
```

Rules:
- `author_id` **MUST** equal the DID referenced by the target `KEY_EVENT`.
- `target_event` **MUST** reference a valid `KEY_EVENT` `packet_id` for that DID.
- The Packet **MUST** be signed by an eligible recovery key; duplicate approvals from the same key **MUST** be ignored.
- `RECOVERY_APPROVAL` Packets **MUST NOT** change key state on their own; they only count toward threshold.
- Threshold approvals **MUST NOT** bypass the recovery delay/dispute window if one is configured.

**Recovery Key Safeguards (RECOMMENDED):**

- **Time-delay on recovery-signed KEY_EVENTs:** Implementations SHOULD treat `KEY_EVENT` Packets signed by a recovery key as pending for a configurable delay window (e.g., 24 hours) before applying them. During this window, clients MAY surface prominent warnings and allow users or other devices to dispute or override the recovery event via additional `KEY_EVENT` Packets signed by non-recovery keys (if any remain valid).

- **Threshold recovery for high-value identities:** Organizations, infrastructure operators, and other high-value identities SHOULD use multi-party or threshold recovery mechanisms (e.g., M-of-N hardware keys, social recovery with trusted contacts) rather than relying on a single recovery key.

- **UX guidance:** Client implementations SHOULD surface the special risk of recovery keys prominently in their UI and encourage users to store recovery keys with stronger protections than ordinary signing keys (e.g., offline storage, safety deposit box, geographically distributed backups).

- **Multiple devices:**
  - A single DID may have multiple authentication keys (one per device).
  All such keys are controlled by the same user.

Recovery mechanisms (social recovery, encrypted backups) are left to client implementations but SHOULD be surfaced in user interfaces.

#### 5.4 Hardware-backed custody (Recommended)

Hardware-backed custody refers to private keys that are generated, stored, and used inside a hardware-backed module where key material is non-exportable (Secure Enclave, StrongBox, TPM, HSM, or a security key).

Guidelines:
- Clients SHOULD prefer non-exportable Ed25519 keys when the platform supports them.
- If a platform cannot sign with Ed25519 inside hardware, clients MAY store the Ed25519 key encrypted with a hardware-bound wrapping key (including passkey-gated access) and require explicit user presence to unwrap for signing.
- Recovery keys SHOULD be hardware-backed or stored offline and MUST NOT be kept on the same device as daily signing keys.
- Clients SHOULD require explicit user presence for high-risk operations (KEY_EVENT, RECOVERY_APPROVAL, identity linking).
- Implementations MUST clearly disclose to the user when a key is software-only versus hardware-backed.

Operators (relays, gates, attestors) SHOULD use HSM-backed signing keys for infrastructure identities to reduce key exfiltration risk and SHOULD maintain audited key-rotation procedures.

### 5.4 Revocation & Validity Windows

- Every verification method in a DID Document **MUST** carry `valid_from` and **SHOULD** carry `valid_until` to bound signature usability.
- The genesis DID Document acts as the bootstrap of the key-event chain: each listed verification method is treated as an implicit `KEY_EVENT` with `event = "ADD"` effective at its `valid_from`. There is **no** validity before `valid_from`; documents omitting `valid_from` for a key are invalid.
- Revocation/rotation events are represented as Packets with `content.type = "KEY_EVENT"`:

```jsonc
{
  "type": "KEY_EVENT",
  "event": "ADD",                 // ADD | ROTATE | REVOKE
  "key_id": "did:strata:alice#keys-2",
  "public_key_multibase": "z6MkfQ...",
  "valid_from": 1715421000,
  "valid_until": 0,               // 0 = open-ended
  "revoked_at": null,             // set when event = REVOKE
  "replaces": "did:strata:alice#keys-1",
  "reason": "device_compromised"
}
```

Rules:
- `KEY_EVENT` Packets **MUST** be signed by an existing non‑revoked key listed in the current DID Document (or by a designated recovery key from that document) and **MUST** be gossiped even if they revoke the signing key.
- The first explicit `KEY_EVENT` after the genesis DID Document MUST chain to an existing verification method whose `valid_from` has already started; implementers MUST NOT assume any implicit earlier validity window.
- Relays **MUST** index `KEY_EVENT` by `author_id` and serve the latest chain to verifiers; clients **MUST** fetch and apply key events before accepting new signatures.
- A Packet signature is valid only if the relay‑observed receive time is within `[valid_from − clock_skew_seconds, min(valid_until, revoked_at) + clock_skew_seconds]` for the signing key (see RFC‑0000 §6.1 for bounds and defaults). Back‑dated `timestamp` fields do **NOT** extend validity.
- Revoked keys **MUST** be treated as unusable for new packets immediately after `revoked_at`; previously received packets remain valid only if they arrived before `revoked_at + clock_skew_seconds`.

### 6. Multiple Personas

A single human may control multiple StrataIDs:

- **Public persona:**
  - Long‑lived DID,
  - Accumulates reputation and trust edges,
  - Used for public posting.

- **Pseudonymous persona(s):**
  - DIDs that do not reveal real‑world identity,
  - May still accumulate reputation within a community.

- **Ephemeral / burner personas:**
  - Short‑lived DIDs,
    - Used for testing, sensitive interactions, or one‑time actions.

The protocol does not attempt to enforce or detect linkages between personas.

### 7. Trust Edges

Trust between identities is expressed via trust edges, which are signed Packets (see RFC‑0002) with `content.type = "TRUST_EDGE"`.

#### 7.1 Trust Edge Content

Minimal trust edge:
```json
{
  "type": "TRUST_EDGE",
  "from": "did:strata:alice",
  "to": "did:strata:bob",
  "strength": 0.8,                    // 0–1
  "context": "friend",                // free-text or enums
  "created_at": 1715421000
}
```

This is embedded in a Packet:

`author_id` MUST equal `from`.

Packet MUST be signed by `from`’s key.

The Packet signature covers the trust edge content; a secondary `edge_signature` field is OPTIONAL and, if present, MUST match the Packet signature scope. `created_at` SHOULD equal the Packet’s `timestamp`; if they differ, clients use the Packet `timestamp` for freshness and deduplication.

#### 7.2 Semantics

- `strength` is a subjective measure (0–1) of how strongly `from` vouches for `to`.
- `context` is descriptive metadata (friend, colleague, source, etc.).
- Clients are free to interpret edges according to their own reputation algorithms (see RFC‑0005).

Trust edges are append‑only; revocations or changes MUST be expressed as new packets, not silent edits.

#### 7.2.1 Inclusion in Trust Graphs

Clients and relays **MUST NOT** include a `TRUST_EDGE` Packet in any trust/reputation computation unless:
- The Packet passes full signature verification per RFC‑0002, and
- `author_id` equals `content.from`, and
- The signing key is currently valid for `author_id` according to RFC‑0001 key‑event rules.

Packets that fail these checks **MUST** be ignored for trust and reputation calculations and **MUST NOT** silently degrade to weaker semantics.

Relays that maintain server‑side indices of trust edges **SHOULD** perform the same checks at ingest time and either reject invalid trust‑edge Packets or store them only in a quarantine bucket that is never surfaced via “trusted edge” APIs.

#### 7.3 Revocation & Updates

Explicit revocation uses `content.type = "TRUST_REVOCATION"`:

```jsonc
{
  "type": "TRUST_REVOCATION",
  "from": "did:strata:alice",
  "to": "did:strata:bob",
  "revokes": ["0xedge1"],        // packet_ids being revoked (optional)
  "reason": "compromised_key",
  "created_at": 1715422000
}
```

Rules:
- `author_id` MUST equal `from`; the Packet MUST be signed by `from`’s non‑revoked key.
- Revocation MUST set the effective strength from `from -> to` to zero for all earlier trust edges with `created_at <= revocation.created_at`. A `strength: 0` `TRUST_EDGE` **is not** a revocation.
- Clients **MUST** ignore attempts to increase strength inside a `TRUST_REVOCATION`.

#### 7.4 Budgets & Abuse Controls

To limit wash‑trading and Sybil inflation:
- Clients and relays **SHOULD** enforce per‑identity trust budgets (see RFC‑0005 §5.2):
  - Default budget: at most 20 edges with `strength > 0.5` per rolling 24h window.
  - Attempts beyond the budget **SHOULD** be rejected or heavily down‑weighted.
- Relays **MUST** rate‑limit `TRUST_EDGE` and `TRUST_REVOCATION` packets per identity to reduce spam/DoS.

> **Note:** This section provides a minimal baseline budget for trust edges. For the complete set of budget parameters—including separate limits for low-strength edges and detailed accounting rules—implementers SHOULD follow RFC-0005 §5.2.

### 8. Security Considerations

- Compromised keys:
  - A compromised key can impersonate an identity until revoked.
  - Clients and relays **MUST** enforce `KEY_EVENT` revocations using relay‑observed time, not author‑supplied timestamps.
- Recovery key compromise:
  - A compromised recovery key can perform complete identity takeover by revoking all other keys and adding attacker-controlled keys.
  - The safeguards in Section 5.3 (time-delay, threshold recovery) are RECOMMENDED to provide defense-in-depth.
  - Users SHOULD treat recovery keys with the same care as cryptocurrency seed phrases or root CA private keys.
- Trust edge abuse:
  - Trust edges reveal aspects of a user’s social graph.
  - Clients **SHOULD** allow users to:
    - Keep some trust edges private,
    - Limit publication of edges by default.
  - Wash‑trading is mitigated via explicit revocation packets and enforced trust budgets.

### 9. Identity Gate Verification Attestations

Identity gates are services that provision StrataIDs. When an identity gate performs verification during provisioning (e.g., email, phone, government ID), it SHOULD issue a verification attestation to record the level of verification performed.

#### 9.1 Verification Attestation Format

```json
{
  "type": "strata.identity.verification",
  "subject_id": "did:strata:z6Mkf...",
  "claims": {
    "verification_level": "phone",
    "verification_method": "sms_otp",
    "verified_at": 1733097600,
    "gate_version": "1.0.0"
  },
  "attestor_id": "did:strata:gate-xyz",
  "attestor_type": "IDENTITY_GATE"
}
```

This attestation is embedded in a Packet signed by the identity gate's key. The `subject_id` is the newly provisioned StrataID.

#### 9.2 Verification Levels

| Level | Description | Example Methods |
|-------|-------------|-----------------|
| `none` | No verification performed | Self-registration |
| `email` | Email address ownership verified | Email link, OTP |
| `phone` | Phone number ownership verified | SMS OTP, voice call |
| `government_id` | Government-issued ID verified | Document scan + liveness |
| `biometric` | Biometric verification | Face match, fingerprint |

Levels are ordered by strength: `none < email < phone < government_id < biometric`.

#### 9.3 Client Interpretation

Clients SHOULD use verification attestations to:
- Adjust Initial Confidence Boost parameters (RFC-0005 §4.5),
- Inform trust decisions about new identities,
- Display verification badges in UI.

Clients MUST validate that:
- The attestation is signed by a valid identity gate key,
- The identity gate's StrataID is trusted (via seed trust, WoT, or explicit configuration),
- The `subject_id` matches the identity being evaluated.

Clients MUST NOT trust verification attestations from unknown or untrusted identity gates.

#### 9.4 Identity Gate Trust

Identity gates MAY be:
- Designated as seed identities in Bootstrap Documents (highest trust),
- Trusted via Web-of-Trust edges from other trusted gates,
- Explicitly configured by the user or client operator.

Clients SHOULD allow users to configure which identity gates they trust for verification attestations. A verification attestation from an untrusted gate MUST be treated as `verification_level: none`.

#### 9.5 Privacy Considerations

- Verification attestations reveal that an identity was provisioned by a specific gate.
- Gates MUST NOT include personally identifiable information (PII) in attestations.
- The `verification_method` field is advisory; it MUST NOT contain raw verification data (phone numbers, ID numbers, etc.).
- Users SHOULD be informed during provisioning that a verification attestation will be published.

### 10. Backwards Compatibility

As new cryptographic primitives or DID resolution mechanisms are introduced:
- Existing DIDs and trust edges **MUST** remain valid.
- New features **SHOULD** be additive and signaled via versioning or feature flags.

### 11. Reference Implementation Notes

- A TypeScript reference library **SHOULD** provide:
  - `createKeypair()`,
  - `createStrataId(publicKey)`,
  - `sign(data, privateKey)`,
  - `verify(signature, data, publicKey)`,
  - Helpers for generating and verifying trust edge Packets.

#### 11.1 Implementation Status (Phase 1)

As of 2025-11-27, the following features from this RFC are implemented:

**strata-identity (Go + TypeScript)**:
- Section 5.4 KEY_EVENT handling: Full implementation of ADD, ROTATE, REVOKE events
  - Go: `key_event.go` with `KeyEventValidator`, `ValidateKeyEvent`, `ApplyKeyEvent`
  - TypeScript: `validateKeyEvent`, `applyKeyEvent`, `createKeyEventContent`
  - Validates authorization rules, key chains, and recovery key restrictions
  - See `/media/sean/SHARED/DEV/strata/strata-identity/README.md` for usage

- Section 5.4 Key validity windows: Full implementation with clock skew tolerance
  - `valid_from`, `valid_until`, `revoked_at` fields on `VerificationMethod`
  - Clock skew tolerance: 300 seconds default, range [60, 300] enforced
  - Go: `IsKeyValidAt(vm, timestamp, clockSkew)` function
  - TypeScript: `isKeyValidAt(vm, timestamp, clockSkew)` function
  - Integrated into packet verification via `VMResolver` interface

- Section 5.3 Recovery key restrictions: Enforced in KEY_EVENT validation
  - Recovery keys (usage: "recovery") can ONLY sign KEY_EVENT or RECOVERY_APPROVAL packets
  - RECOVERY_APPROVAL packets must be signed by recovery keys
  - `IsRecoveryKeyAuthorized(vm, contentType)` checks in Go and TypeScript

**strata-relay (Go)**:
- Section 5.4 Signature verification: Enabled by default per RFC-0004
  - `verification.enabled: true` is default configuration
  - Ed25519 signature verification before PUBLISH acceptance
  - Key validity window checks using relay-observed receive time
  - Error codes: `invalid_signature`, `key_expired`, `key_revoked`, `key_not_yet_valid`
  - See `/media/sean/SHARED/DEV/strata/strata-relay/README.md` for configuration

**Pending**:
- Section 3.3 DID resolution from network relays (currently bootstrap-file only)
- Section 7 Trust edge indexing and validation (server-side in relays)
- Multi-device key management UX (client implementation)
