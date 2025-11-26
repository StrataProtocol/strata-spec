# RFC 0000 – Terminology and Conventions

- Status: Draft
- Author: Sean Red
- Created: 2025-11-25
- Updated: 2025-11-26
- **Scope:** Normative conventions (terminology, canonicalization, identifiers)

> **BREAKING CHANGE (2025-11-26):** Wire encoding for content-addressed identifiers (`packet_id`, `genesis_id`, `stake_id`, `bootstrap_id`) changed from multibase (base58btc) to `0x` + lowercase hex. Multibase `z` prefix is now reserved exclusively for DIDs and DID-related fields. See 5.2 and 5.4.


## 1. Summary

This RFC defines shared **terminology**, **notation**, and **conventions** used across all Strata specifications:

- Core protocol terms (StrataID, Packet, Relay, etc.),
- JSON and canonicalization rules,
- Time and identifier formats,
- Use of normative language (**MUST**, **SHOULD**, **MAY**, etc.).

It is intended to be read before other RFCs.

## 2. Normative Language

The key words:

- **MUST**,  
- **MUST NOT**,  
- **REQUIRED**,  
- **SHALL**,  
- **SHALL NOT**,  
- **SHOULD**,  
- **SHOULD NOT**,  
- **RECOMMENDED**,  
- **MAY**,  
- **OPTIONAL**

in Strata RFCs are to be interpreted as described in [RFC 2119].

Where there is a conflict between RFC 0000 and another RFC, the more specific RFC for that feature takes precedence, but SHOULD reference this document’s conventions.

---

## 3. Core Terminology

### 3.1 StrataID

- A **StrataID** is a decentralized identifier (DID) for an identity in the Strata Protocol.
- It uses the DID method **`strata`**:
  ```text
  did:strata:<method-specific-id>
  ```

- It is generally backed by one or more public/private keypairs.

Examples:
- `did:strata:z6MkfQ...` (user identity),
- `did:strata:relay_1` (relay identity),
- `did:device:pixel_10_abc` (device identity – non‑strata DID method, referenced by Strata).

When this spec says “identity” without qualification, it usually refers to an entity identified by a StrataID.

### 3.2 Packet

A **Packet** is the atomic data unit in Strata.
- It is a signed JSON object containing:
  - Metadata (`author`, `timestamp`, `version`),
  - `content` (`post`, `message`, `trust edge`, `attestation`, etc.),
  - `provenance_header` (for media),
  - `attestations` (optional),
  - `consensus_metrics` (optional),
  - `packet_id` and `signature`.

Packets are tamper‑evident and can be validated using the `author_id`’s public key.

### 3.3 Relay

A Relay is a network node that:
- Accepts Packets from clients,
- Stores them (for some time),
- Forwards them to subscribers according to filters.

Relays are content‑agnostic at the protocol level and MUST NOT modify Packets.
Relays may enforce local policies (rate limits, blocking, etc.) but are not authorities on truth.

### 3.4 Client

A Client is a user‑facing application that:
- Holds and uses keys for one or more StrataIDs,
- Connects to one or more Relays,
- Displays a feed or other UI around Packets,
- Runs the Trust Engine / Reality Tuner (local filtering and labeling logic).

Examples:
- Reference Strata PWA,
- Third‑party social app integrating Strata,
- Research tooling for provenance inspection.

### 3.5 Trust Engine (Reality Tuner) & Reality Switch

The **Trust Engine** (formerly nicknamed the Reality Tuner) is the client‑side view filter—not a truth oracle—that combines:
- Packet content,
- Provenance,
- Attestations,
- Reputation,
- User preferences
into a feed and labels.

The Reality Switch is a high‑level UX control for selecting coarse policy profiles:
- Strict (Grandma),
- Standard (Default),
- Wild (Developer).
Attestations are defined in RFC‑0003; Trust Engine and Reality Switch behavior is specified in RFC‑0005 (with future tuning RFCs extending thresholds).

### 3.6 Genesis Event
A Genesis Event is a record that describes how a media artifact first entered Strata.
It binds:
- `media_hash`,
- `origin_type` (`HARDWARE_SECURE_ENCLAVE`, `AI_MODEL`, `SOFTWARE`, `UNKNOWN`),
- Origin metadata (device/model details, timestamps),
- Issuer signatures.

Packets reference Genesis Events via `genesis_media_hash` or `genesis_id`.

### 3.7 Provenance Header

The `provenance_header` is a field in a Packet that summarizes provenance for the media referenced by that Packet:
- `origin_type` (`HARDWARE_SECURE_ENCLAVE`, `AI_MODEL`, `SOFTWARE`, `UNKNOWN`),
- `genesis_media_hash`,
- `device_id` / `device_signature` (optionally),
- `edit_history` (list of transformations).

It is not itself a Genesis Event, but may refer to one.

### 3.8 Attestation

An Attestation is a signed claim about a Packet.
- Attestations may be:
  - Embedded inside the Packet (`packet.attestations`),
  - Broadcast as standalone Packets (`content.type = "ATTESTATION"`).

Typical claims:
- Provenance analysis (synthetic vs human),
- Manipulation (e.g., “cropped to mislead”),
- Fact‑checking (e.g., “caption misleading”)
- Spam/abuse classification.

### 3.9 Trust Edge

A Trust Edge is a signed statement where identity A vouches for identity B with some strength and context.

Trust edges form the **Web‑of‑Trust** graph used in reputation calculations.
Defined in detail in RFC 0001.

### 3.10 Stake
- Stake is locked value backing a beneficiary identity or claim, used as an anti‑Sybil/message‑flooding primitive.
- Stake entries and slashing events are represented as Packets.

How stake is economically implemented (on‑chain/off‑chain) is beyond the protocol’s core scope.

## 4. JSON & Canonicalization
Strata’s core data structures use JSON. To avoid ambiguity:

### 4.1 JSON Requirements

- No comments in actual protocol data (examples in RFCs may include // comments, but wire data **MUST NOT**).
- UTF‑8 encoding.
- Keys are case‑sensitive.
- Unknown fields **MUST** be ignored, not cause errors (unless otherwise specified for a particular structure).

### 4.2 Canonicalization

For hashing and signing operations (e.g., `computing packet_id`) :
- Objects **MUST** be canonicalized in a deterministic way, such as JSON Canonicalization Scheme (JCS), including:
  - Sorted keys lexicographically,
  - No trailing commas,
  - No insignificant whitespace.
- Implementations **MUST** serialize in the same way when computing hashes and signatures.
- The canonicalization algorithm used **MUST** be documented and kept stable across versions of a given client/library.

## 5. Hashing & Identifiers
### 5.1 Hash Functions

For `packet_id`, `genesis_id`, and other content‑addressed identifiers, implementations **MUST** use `BLAKE3-256`. SHA‑256 MAY be accepted for legacy ingestion when explicitly indicated, but it MUST NOT be emitted by conformant implementations.

### 5.2 `packet_id`
`packet_id` is a string representing a hash of the canonical Packet body (excluding `packet_id` and `signature`).

Encoding:
- Wire format for v0 **MUST** be `0x` followed by the lowercase hex encoding of the raw digest bytes (BLAKE3-256).
- Implementations **MUST NOT** emit base58btc (multibase `z`) for `packet_id` on the wire.
- Example: `"0xaf1349b9f5f9a1a6a0404dea36dcc9499bcb25c9adc112b7cc9a93cae41f3262"` (32-byte BLAKE3-256 digest, hex encoded with `0x` prefix).

### 5.3 `genesis_id`, `bootstrap_id`, `stake_id`
Other content-addressed identifiers (e.g., `genesis_id`, `bootstrap_id`, `stake_id`) follow the same principle:
- Derived from a canonical hash of the underlying object,
- Wire format **MUST** be `0x` + lowercase hex of the digest bytes,
- Implementations **MUST NOT** emit base58btc for these identifiers.

### 5.4 Encoding Distinction: DIDs vs Content IDs

Strata uses two distinct encoding schemes:

| Identifier Type | Wire Encoding | Example |
|-----------------|---------------|---------|
| DIDs and DID-related fields (`author_id`, `attestor_id`, `device_id`, etc.) | Multibase `z` (base58btc) per DID spec | `did:strata:z6MkfQ...` |
| Content-addressed IDs (`packet_id`, `genesis_id`, `stake_id`, `bootstrap_id`) | `0x` + lowercase hex | `0x1e20a1b2c3d4...` |

Rationale:
- DIDs follow W3C DID Core conventions which use multibase for method-specific identifiers.
- Content-addressed IDs use hex for debuggability, tooling compatibility, and alignment with Ethereum/IPFS ecosystem conventions.

## 6. Time & Timestamps

- All timestamps in Strata structures **MUST** use Unix epoch seconds (integer, UTC).
- RFCs sometimes use descriptive names like `created_at`, `issued_at`, `capture_timestamp`; these all represent epoch seconds.
- Clients MAY display localized times to users, but MUST store and process epoch seconds on the wire.

### 6.1 Clock Skew Tolerance

- Strata uses a shared tolerance parameter `clock_skew_seconds` for time-window checks (e.g., signature validity, freshness).
- **Default:** `clock_skew_seconds = 300` (5 minutes).
- **Bounds:** Implementations **MUST** keep `clock_skew_seconds` within `[60, 300]` seconds for interoperability; values outside this range **MUST** be ignored in favor of the default.
- **Overrides:** Bootstrap documents and relay manifests **MAY** publish `clock_skew_seconds`; clients/relays **MUST** converge on the same value by clamping to the bounds above and preferring bootstrap configuration over local defaults.
- All references to `clock_skew` or `clock_skew_seconds` in other RFCs refer to this parameter unless explicitly stated otherwise.

## 7. Enumerations & String Constants
Strata frequently uses string constants for enums. Unless otherwise specified:
- Enums are case‑sensitive.
- Unknown enum values:
  - **MUST NOT** cause a hard error,
  - **SHOULD** result in “`unknown` / fallback” behavior in clients.

### 7.1 `origin_type`
Values (initial):
- `HARDWARE_SECURE_ENCLAVE`
- `AI_MODEL`
- `SOFTWARE`
- `UNKNOWN`

### 7.2 `attestor_type`
Values (initial):
- `CLIENT`
- `NGO`
- `LAB`
- `MEDIA`
- `MODEL_PROVIDER`
- `OTHER`

### 7.3 `content.type`

Values (initial):
- `POST`
- `MESSAGE`
- `TRUST_EDGE`
- `TRUST_REVOCATION`
- `ATTESTATION`
- `CONFIG`
- `STAKE`
- `STAKE_SLASH`
- `KEY_EVENT`
- `CONSENSUS_METRIC`
- `CORRECTION`
- Additional types may be defined by future RFCs.

## 8. Security & Privacy Notes
- JSON bodies and media payloads from untrusted sources **MUST** be treated as untrusted input.
- Cryptographic verification (signature + hash) **MUST** precede trust.
- Provenance headers and attestations provide evidence, not guarantees.
- Implementations **SHOULD** minimize logging of sensitive metadata (IP addresses, cross‑relay correlation data) to reduce surveillance risk.

## 9. Versioning
### 9.1 RFC Versioning
- Each RFC defines a major behavior or structure.
- New RFCs or revisions should:
- Be numbered sequentially,
- Clearly mark BREAKING CHANGES where applicable.

### 9.2 Protocol Version Fields

- Many structures include a version field (e.g., Packet version).
- Consumers **MUST**:
    - Check version,
    - Reject or treat unknown versions with caution,
    - Provide clear error handling for unsupported versions.

## 10. References
- [RFC 2119] – Key words for use in RFCs to Indicate Requirement Levels.
- [RFC 0001-tbd] – Identity and StrataID.
- [RFC 0002-tbd] – Data Packets and Provenance.
- [RFC 0003-tbd] – Attestations and Retroactive Consensus.
- [RFC 0004-tbd] – Relays, Bootstrap, and Discovery.
