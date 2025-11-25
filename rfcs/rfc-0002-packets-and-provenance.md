<!-- rfcs/rfc-0002-packets-and-provenance.md -->

# RFC-0002: Data Packets and Provenance Headers

- **Status:** Draft  
- **Author(s):** Strata Core Team  
- **Created:** 2025-11-25  
- **Updated:** 2025-11-25  

---

## 1. Summary

This RFC defines:

- The **Data Packet** structure, Strata’s atomic data unit,
- Canonicalization and packet ID computation,
- The **provenance_header** field,
- The notion of **Genesis Events** for media.

---

## 2. Motivation

Strata “thinks in Packets, not posts.” Every action—post, message, attestation, trust edge—is represented as a signed Packet.

Requirements:

- Tamper‑evident: changes to content or metadata must invalidate signatures.
- Extensible: new content types and metadata fields can be added without breaking interoperability.
- Provenance‑aware: Packets must be able to reference origin (capture or generation) of media.

---

## 3. Packet Structure

### 3.1 Top‑Level Packet

A Packet is a JSON object with the following top‑level fields:

```jsonc
{
  "packet_id": "0x8f2...a1",
  "version": 1,
  "timestamp": 1715421200,
  "author_id": "did:strata:zero_cool",

  "content": { ... },
  "provenance_header": { ... },
  "attestations": [ ... ],
  "consensus_metrics": { ... },

  "signature": "0xFinalPacketSignature..."
}

Fields:
- `packet_id` – unique identifier derived from a hash of the canonicalized Packet.
- `version` – protocol version of this Packet schema.
- `timestamp` – Unix epoch seconds when the packet was created.
- `author_id` – StrataID (DID) of the author.
- `content` – content payload (see below).
- `provenance_header` – provenance information (optional for non‑media content).
- `attestations` – embedded attestations (see RFC‑0003) (optional).
- `consensus_metrics` – optional aggregate counters.
- `signature` – author’s signature over the Packet.
```

### 3.2 Content

Minimal content object:
```jsonc
{
  "type": "POST",                 // POST | MESSAGE | TRUST_EDGE | ATTESTATION | CONFIG | ...
  "text": "Consensus node deployed.",
  "media": [
    {
      "media_hash": "ipfs://QmHash...",
      "media_type": "image/jpeg",
      "size_bytes": 238472
    }
  ],
  "thread_ref": null              // optional, reference to another packet_id
}
```

Fields:
- `type` – application‑level content type.
- `text` – optional text body.
- `media` – array of media references (optional).
- `thread_ref` – optional packet_id for replies, threads, etc.

Content types are extensible. Clients **SHOULD** gracefully ignore unknown types.

## 4. Canonicalization and Packet ID
### 4.1 Canonicalization

To avoid signature ambiguity, Packets **MUST** be canonicalized before hashing and signing:
- Keys sorted lexicographically at each object level,
- No comments,
- No trailing commas,
- UTF‑8 encoding.

A canonical JSON representation (e.g. JSON Canonicalization Scheme) **SHOULD** be used and referenced in spec implementations.

### 4.2 Packet ID & Signature
- `packet_id` is computed as:
  ```
  packet_id = hash(canonicalized_packet_without_packet_id_and_signature)
  ```
- `signature` is computed as:
  ```
  signature = sign(canonicalized_packet_without_packet_id_and_signature, author_private_key)
  ```

Verification:
1. Re‑canonicalize the Packet with `packet_id` and `signature` removed.
2. Recompute `packet_id` and compare.
3. Verify `signature` against the author’s public key.

## 5. Provenance Header
The provenance_header encodes how media attached to a Packet was created and transformed.

```jsonc
"provenance_header": {
  "origin_type": "HARDWARE_SECURE_ENCLAVE",
  // HARDWARE_SECURE_ENCLAVE | AI_MODEL | SOFTWARE | UNKNOWN

  "genesis_media_hash": "ipfs://QmGenesisHash...",
  "device_id": "did:device:pixel_10_abcd",
  "device_signature": "0xSig...",
  "edit_history": [
    {
      "action": "CROP",
      "software_id": "did:software:Adobe_C2PA_Plugin",
      "timestamp": 1715421100,
      "signature": "0xEditSig..."
    }
  ]
}
```

Fields:
- `origin_type`:
  - `HARDWARE_SECURE_ENCLAVE` – hardware‑signed capture,
  - `AI_MODEL` – AI‑generated media,
  - `SOFTWARE` – software‑generated/edited,
  - `UNKNOWN` – legacy or opaque origin.
- `genesis_media_hash`:
  - Content hash of the original media at creation time.
- `device_id`:
  - DID of the capturing device or device class (optional, for hardware origin).
- `device_signature`:
  - Signature by the device or device class key, attesting to the genesis hash and capture metadata.
- `edit_history`:
  - Ordered list of transformations applied after genesis.
  - Each entry includes:
    - `action` – description or enum (CROP, RESIZE, ENCODE, etc.),
    - `software_id` – DID of the software performing the edit (optional),
    - `timestamp` – time of edit,
    - `signature` – optional signature by the software or user key.

Provenance headers MAY be omitted for non‑media or trivial content; clients SHOULD then treat provenance as UNKNOWN.

## 6. Genesis Events
A Genesis Event is a standalone object that records the initial origin of specific media.
```jsonc
{
  "genesis_id": "0xgenesis123",
  "media_hash": "ipfs://QmGenesisHash...",
  "origin_type": "HARDWARE_SECURE_ENCLAVE",
  "details": {
    "hardware": {
      "device_model": "camera-vendor-x-model-y",
      "device_public_key": "0xDevKey...",
      "capture_timestamp": 1715421000,
      "hardware_signature": "0xHwSig..."
    },
    "ai_model": null,
    "software": null
  },
  "created_at": 1715421005,
  "issuer": "did:strata:device_vendor_or_model_provider",
  "signature": "0xGenesisIssuerSig..."
}
```

Fields:
- `genesis_id`:
  - Unique identifier for this Genesis Event.
- `media_hash`:
  - Content hash of the original media at creation time.
- `origin_type`:
  - `HARDWARE_SECURE_ENCLAVE` – hardware‑signed capture,
  - `AI_MODEL` – AI‑generated media,
  - `SOFTWARE` – software‑generated/edited,
  - `UNKNOWN` – legacy or opaque origin.
- `details`:
  - Hardware details (for `HARDWARE_SECURE_ENCLAVE`),
  - AI model details (for `AI_MODEL`),
  - Software details (for `SOFTWARE`).
- `created_at`:
  - Unix epoch seconds when the Genesis Event was created.
- `issuer`:
  - DID of the issuer of the Genesis Event.
- `signature`:
  - Signature by the issuer, attesting to the Genesis Event.

Genesis Events are referenced by genesis_media_hash or genesis_id from Packets.
They allow independent verification of hardware/model claims.
AI‑generated genesis events (origin_type = "AI_MODEL") populate details.ai_model instead.

## 7. Consensus Metrics
consensus_metrics is an optional field for aggregated counts:
```jsonc
"consensus_metrics": {
  "human_votes": 450,
  "synth_votes": 12,
  "spam_stake": 0
}
```

Fields:
- `human_votes`:
  - Number of users marking content as “likely human/real.”
- `synth_votes`:
  - Marking as “likely synthetic.”
- `spam_stake`:
  - Stake pledged to assert non‑spam nature of content.
These are hints, not truth. Their computation and update are outside the scope of this RFC and may be implementation‑defined.

## 8. Security Considerations
- Provenance headers:
  - Only as trustworthy as the issuers of device/model keys.
  - May be forged if hardware keys are compromised.
- Edit histories:
  - Rely on compliant software; malicious editors can omit edits.
- Genesis Events:
  - Should be signed by reputable issuers (hardware vendors, model providers).

Clients **MUST** treat provenance data as **evidence**, not as guarantees of truth.