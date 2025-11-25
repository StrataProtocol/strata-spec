<!-- rfcs/rfc-0001-identity-and-strataid.md -->

# RFC-0001: Identity and StrataID

- **Status:** Draft  
- **Author(s):** Strata Core Team  
- **Created:** 2025-11-25  
- **Updated:** 2025-11-25  

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
  "@context": "https://www.w3.org/ns/did/v1",
  "id": "did:strata:z6MkfQ...",
  "verificationMethod": [
    {
      "id": "did:strata:z6MkfQ#keys-1",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:strata:z6MkfQ",
      "publicKeyMultibase": "z6MkfQ..."
    }
  ],
  "authentication": [
    "did:strata:z6MkfQ#keys-1"
  ]
}
```
- It **MAY** include:
  - Additional keys (for rotation or multi‑device),
  - Service endpoints (e.g., preferred relays),
  - Metadata (e.g., human‑readable name).

Strata does not mandate a specific DID Document registry; DID Documents can be:

- Stored via content addressing and resolved through relays,
- Anchored in external systems (blockchains, key servers),
- Distributed through bootstrap documents.

### 3.3 DID Resolution & Distribution

Clients **MUST** follow a deterministic resolution order to avoid split‑brain identity views:

1. Check locally cached DID Document referenced by its multihash (`did_doc_hash`) seen within a freshness window.
2. Query bootstrap documents (whitepaper §10.1) and relay manifests for `did_doc_hash` pointers for the target DID.
3. Fetch the content‑addressed DID Document by `did_doc_hash` from relays or other stores.

Rules:
- DID Documents **MUST** be content‑addressed; `did_doc_hash = multihash(blake3-256(canonical_did_doc))`.
- Updates **MUST** be monotonic and linked via `prev_did_doc_hash` to allow replay/rollback protection.
- If multiple heads exist for the same DID (conflict), clients **MUST** fail closed and surface the conflict instead of guessing.
- Relay operators **SHOULD** gossip `did_doc_hash` updates alongside identity‑scoped packets to speed propagation.

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

- **Multiple devices:**
  - A single DID may have multiple authentication keys (one per device).
  All such keys are controlled by the same user.

Recovery mechanisms (social recovery, encrypted backups) are left to client implementations but SHOULD be surfaced in user interfaces.

### 5.4 Revocation & Validity Windows

- Every verification method in a DID Document **MUST** carry `valid_from` and **SHOULD** carry `valid_until` to bound signature usability.
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
- `KEY_EVENT` Packets **MUST** be signed by an existing non‑revoked key (or a designated offline recovery key) and **MUST** be gossiped even if they revoke the signing key.
- Relays **MUST** index `KEY_EVENT` by `author_id` and serve the latest chain to verifiers; clients **MUST** fetch and apply key events before accepting new signatures.
- A Packet signature is valid only if the relay‑observed receive time is within `[valid_from − clock_skew, min(valid_until, revoked_at) + clock_skew]` for the signing key. Back‑dated `timestamp` fields do **NOT** extend validity.
- Revoked keys **MUST** be treated as unusable for new packets immediately after `revoked_at`; previously received packets remain valid only if they arrived before `revoked_at + clock_skew`.

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

### 8. Security Considerations

- Compromised keys:
  - A compromised key can impersonate an identity until revoked.
  - Clients and relays **MUST** enforce `KEY_EVENT` revocations using relay‑observed time, not author‑supplied timestamps.
- Trust edge abuse:
  - Trust edges reveal aspects of a user’s social graph.
  - Clients **SHOULD** allow users to:
    - Keep some trust edges private,
    - Limit publication of edges by default.
  - Wash‑trading is mitigated via explicit revocation packets and enforced trust budgets.

### 9. Backwards Compatibility

As new cryptographic primitives or DID resolution mechanisms are introduced:
- Existing DIDs and trust edges **MUST** remain valid.
- New features **SHOULD** be additive and signaled via versioning or feature flags.

### 10. Reference Implementation Notes

- A TypeScript reference library **SHOULD** provide:
  - `createKeypair()`,
  - `createStrataId(publicKey)`,
  - `sign(data, privateKey)`,
  - `verify(signature, data, publicKey)`,
  - Helpers for generating and verifying trust edge Packets.
