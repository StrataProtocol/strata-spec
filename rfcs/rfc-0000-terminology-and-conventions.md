# RFC 0000 – Terminology and Conventions

- Status: Draft
- Author: Sean Red
- Created: 2025-11-25
- Updated: 2025-11-25


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
- Runs the Reality Tuner (local filtering and labeling logic).

Examples:
- Reference Strata PWA,
- Third‑party social app integrating Strata,
- Research tooling for provenance inspection.

### 3.5 Reality Tuner & Reality Switch

The Reality Tuner is the client‑side policy engine that combines:
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
Exact implementation details are defined in RFC 0003 and future tuning RFCs.

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

For `packet_id` and content addressing, implementations **SHOULD** use a strong cryptographic hash (e.g., SHA‑256 or BLAKE3).

When using multihash/multibase, the hash function must be indicated explicitly.

### 5.2 `packet_id`
`packet_id` is a string representing a hash of the canonical Packet body (excluding `packet_id` and `signature`).

Common encoding:
- Hex string prefixed with 0x,
- OR multibase/multihash encoding (recommended long‑term).

### 5.3 `genesis_id`, `bootstrap_id`, etc.
Other identifiers (e.g., `genesis_id`, `bootstrap_id`, `stake_id`) follow the same principle:
- Derived from a canonical hash of the underlying object,
- Encoded as hex or multibase.

## 6. Time & Timestamps

- All timestamps in Strata structures **MUST** use Unix epoch seconds (integer, UTC).
- RFCs sometimes use descriptive names like `created_at`, `issued_at`, `capture_timestamp`; these all represent epoch seconds.
- Clients MAY display localized times to users, but MUST store and process epoch seconds on the wire.

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
- `ATTESTATION`
- `CONFIG`
- `STAKE`
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
- [RFC 0003-tbd] – Attestations and Reality Tuner.
- [RFC 0004-tbd] – Relays, Bootstrap, and Discovery.