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
  "packet_id": "zb7...a1",
  "version": 1,
  "timestamp": 1715421200,
  "author_id": "did:strata:zero_cool",
  "expires_at": 1715500000,          // optional absolute TTL
  "nonce": "b64random...",            // optional replay guard

  "content": { ... },
  "provenance_header": { ... },
  "attestations": [ ... ],
  "consensus_metrics": { ... },

  "signature": "0xFinalPacketSignature..."
}

Fields:
- `packet_id` – unique identifier derived from a hash of the canonicalized Packet.
- `version` – protocol version of this Packet schema.
- `timestamp` – Unix epoch seconds when the packet was created (author‑provided).
- `expires_at` – optional Unix epoch seconds after which relays **MUST NOT** serve the Packet.
- `nonce` – optional high‑entropy string to aid replay detection across relays.
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

### 3.3 Freshness, Timestamps, and Relay Witnessing

- Authors provide `timestamp`, but relays **MUST** apply freshness checks:
  - Reject Packets with `timestamp` more than **5 minutes** in the future relative to relay clock.
  - Reject Packets with `timestamp` older than **24 hours** unless explicitly configured for historical backfill.
- Relays **MUST** record `received_at` (their own clock) for each accepted Packet and include it when serving/subscribing (see RFC‑0004). Clients **MUST** prefer `received_at` for quorum/age calculations and only fall back to author timestamps if `received_at` is unavailable.
- If `expires_at` is set, relays **MUST** stop serving the Packet after `expires_at`. Clients **MAY** delete cached ciphertext/media after that time.
- `nonce` is OPTIONAL and can be used by relays to detect cross‑relay replays of the same Packet body prior to `packet_id` computation collisions.

### 3.4 Corrections and Retractions

Strata has no hard delete. To correct or retract information:
- Authors **MUST** publish a new Packet with `content.type = "CORRECTION"` referencing `target_packet`.
- `CORRECTION` payload may include replacement text/media and an optional `reason` string.
- Attestors retracting a mistaken attestation **SHOULD** emit a `CORRECTION` Packet that supersedes the earlier attestation by ID.
- Clients **SHOULD** surface the correction link prominently and down‑rank or blur superseded Packets but MUST retain original history for auditability.

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
  packet_id = multibase(multihash(blake3-256(
      canonicalized_packet_without_packet_id_and_signature)))
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
      "prev_edit_hash": "0xprev",
      "resulting_media_hash": "ipfs://QmAfterCrop...",
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
    - `prev_edit_hash` – hash of the previous edit entry or the genesis media hash,
    - `resulting_media_hash` – hash of media after the edit,
    - `signature` – REQUIRED for `origin_type = HARDWARE_SECURE_ENCLAVE` to maintain green‑ring status; SHOULD be present otherwise.

Rules:
- For `HARDWARE_SECURE_ENCLAVE` and `AI_MODEL`, `edit_history` **MUST** form a hash chain from genesis to the current media reference with no gaps; missing or broken chains downgrade provenance to `UNKNOWN/YELLOW`.
- Provenance headers MAY be omitted for non‑media or trivial content; clients SHOULD then treat provenance as UNKNOWN.

### 5.1 Media Storage & Availability

- Media referenced by `media_hash` **MUST** be content‑addressed (e.g., IPFS/Arweave hash).
- Responsibility for persistence:
  - Authors **SHOULD** pin or otherwise ensure availability of referenced media.
  - Relays **MAY** cache/pin but are not required to store media blobs.
  - Bootstrap documents may list recommended pinning services; clients **SHOULD** warn when media is unpinned/unavailable.
- If media cannot be fetched or its hash mismatches, clients **MUST** downgrade provenance to `UNKNOWN` for that media reference.

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
  "issuer_chain": [
    {
      "subject": "did:device:pixel_10_abcd",
      "issuer": "did:manufacturer:vendor_x",
      "certificate": "base64url(...)"
    },
    {
      "subject": "did:manufacturer:vendor_x",
      "issuer": "did:hardware-root:2025",
      "certificate": "base64url(...)"
    }
  ],
  "attestation_evidence": {
    "format": "android-key-attestation",
    "payload": "base64url(...)",
    "nonce": "base64url(...)"
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
- `issuer_chain`:
  - Ordered certificates/attestations anchoring `device_public_key` (or model key) to a manufacturer/provider root.
- `attestation_evidence`:
  - Structured evidence (e.g., Android/Apple hardware attestation, C2PA assertion) binding device/model key to hardware/model and nonce.

Genesis Events are referenced by genesis_media_hash or genesis_id from Packets.
They allow independent verification of hardware/model claims.

Verification requirements:
- For `origin_type = HARDWARE_SECURE_ENCLAVE`, Genesis Events **MUST** include `issuer_chain` that chains to hardware manufacturer roots distributed in bootstrap documents; clients **MUST** validate the chain and attestation evidence. If missing or invalid, provenance for that media **MUST** be treated as `UNKNOWN/SOFTWARE`.
- For `origin_type = AI_MODEL`, `issuer_chain` anchors the model/provider key; clients **MUST** verify the chain against model provider roots from bootstrap documents or local policy.
- `device_public_key` and model keys are authenticated via the validated chain; unsigned or self‑asserted keys are insufficient for green‑ring treatment.

## 7. Consensus Metrics

The in‑packet `consensus_metrics` field is **RESERVED**. Authors **MUST NOT** populate it; clients **MUST** ignore unsigned values.

Aggregated counters **MUST** be published as standalone Packets with `content.type = "CONSENSUS_METRIC"`:

```jsonc
{
  "content": {
    "type": "CONSENSUS_METRIC",
    "target_packet": "zb7...a1",
    "metrics": {
      "human_votes": 450,
      "synth_votes": 12,
      "spam_stake": 0
    },
    "source_id": "did:strata:relay_eu1",
    "observed_at": 1715423000
  },
  "signature": "0xSourceSignature..."
}
```

Rules:
- `source_id` identifies the relay/aggregator computing the metrics.
- The Packet signature by `source_id` **MUST** cover `target_packet` and `metrics`.
- Clients **MUST** treat metrics as advisory evidence tied to the source identity’s reputation, not as authoritative truth.

## 8. Replay Protection and Deduplication

- Relays **MUST** maintain a de‑duplication cache of `packet_id` for at least the maximum allowed backfill window (default 7 days) and drop duplicate submissions of already‑accepted Packets.
- Relays **MUST** reject Packets older than the configured backfill window (default 24 hours) unless a dedicated historical import path is used.
- For attestation and consensus metric packets, `attestation_id`/`content.target_packet` pairs **MUST** be de‑duplicated to avoid replay‑triggered quorum gaming.
- Clients **SHOULD** treat multiple deliveries of the same `packet_id` as the same logical Packet; repeated receipt MUST NOT increase counters or reputation.

## 9. Security Considerations
- Provenance headers:
  - Only as trustworthy as the issuers of device/model keys.
  - May be forged if hardware keys are compromised.
- Edit histories:
  - Rely on compliant software; malicious editors can omit edits (which downgrades provenance per §5).
- Timestamp games:
  - Freshness rules in §3.3 MUST be enforced; `received_at` (relay‑observed) time is authoritative for replay/quorum calculations, not author‑supplied timestamps.
- Genesis Events:
  - Should be signed by reputable issuers (hardware vendors, model providers) and chained to known roots.

Clients **MUST** treat provenance data as **evidence**, not as guarantees of truth.
