<!-- rfcs/rfc-0002-packets-and-provenance.md -->

# RFC-0002: Data Packets and Provenance Headers

- **Status:** Draft
- **Author(s):** Strata Core Team
- **Created:** 2025-11-25
- **Updated:** 2025-11-28
- **Scope:** Normative protocol (Packet wire format, provenance headers, genesis)

> **BREAKING CHANGE (2025-11-26):** `packet_id` and `genesis_id` wire encoding changed from multibase (base58btc) to `0x` + lowercase hex. See RFC-0000 5.2-5.4.  

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
  "packet_id": "0x1e208f2c3a4b5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f7a1",
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
- `provenance_header` – provenance information. **MUST** be present when the Packet references media (e.g., `content.media` non-empty); MAY be omitted for non‑media content.
- `attestations` – embedded attestations (see RFC‑0003) (optional). Each embedded attestation **MUST** include a `target_packet` field matching this Packet's `packet_id`; see RFC‑0003 §3.4 for signature and validation rules. The optional top-level `attestations` array is reserved for Attestation objects as defined in RFC‑0003. Clients SHOULD treat these as "embedded attestations" and handle them identically to standalone attestation Packets once validated.
- `consensus_metrics` – optional aggregate counters.
- `signature` – author’s signature over the Packet.
```

### 3.2 Content

Minimal content object:
```jsonc
{
  "type": "POST",                 // POST | MESSAGE | TRUST_EDGE | ATTESTATION | ATTESTATION_RETRACTION | CORRECTION | CONFIG | ...
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

When a client receives the same `packet_id` from multiple relays with different `received_at` values, it MUST use the earliest `received_at` for all age-based and quorum calculations. Implementations MAY retain per-relay `received_at` values for debugging or advanced analysis, but MUST use a deterministic rule (earliest is REQUIRED) for security-sensitive logic including key validity windows, retroactive consensus timing, and trust budget accounting.

### 3.4 Corrections and Retractions

Strata has no hard delete. To correct or retract information:
- Authors **MUST** publish a new Packet with `content.type = "CORRECTION"` referencing `target_packet`.
- Only the original author of `target_packet` (or the original attestor, if `target_packet` is an attestation packet) **MAY** issue a superseding correction; clients/relays **MUST** ignore supersession attempts from other authors.
- `CORRECTION` payload **MUST** include `action: "replace" | "retract"` plus an optional `reason`; `replace` may carry replacement text/media, while `retract` signals that the original should be treated as withdrawn without substitute content.
- Ordering: if multiple valid corrections exist from the original author/attestor, the latest by relay‑observed `received_at` **MUST** win (tie‑break by lexicographically smallest `packet_id`).
- Attestors retracting a mistaken attestation **SHOULD** emit either a `CORRECTION` with `action: "retract"` or an `ATTESTATION_RETRACTION` Packet (see RFC‑0003); either form **MUST** zero the attestation’s weight in quorum calculations.
- Clients **SHOULD** surface the correction link prominently and down‑rank or blur superseded Packets but MUST retain original history for auditability.

### 3.5 Packet Size Constraints

Implementations MUST enforce a maximum size for the canonical JSON representation of a Packet (excluding transport framing).

- **RECOMMENDED default:** 256 KiB (262,144 bytes)
- **Hard limit:** Implementations SHOULD NOT accept Packets larger than 1 MiB; values exceeding this are strongly discouraged and MAY be rejected without further processing.
- Large media MUST be stored out-of-band and referenced via `media_hash` (see §5.1); the Packet itself should contain only metadata and references.

Clients SHOULD NOT construct Packets whose canonical JSON exceeds 256 KiB.

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
  packet_id = "0x" + hex(blake3-256(
      canonicalized_packet_without_packet_id_and_signature))
  ```
  where `hex()` produces lowercase hexadecimal encoding.
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
- Packets that reference media **MUST** include a `provenance_header`. For non‑media or trivial content, the header MAY be omitted; clients SHOULD then treat provenance as UNKNOWN.

### 5.0 Minimum Provenance Header (Unknown)

When origin is unknown or no richer provenance is available, Packets that include media **MUST** still supply the minimal header:

```jsonc
{
  "origin_type": "UNKNOWN",
  "genesis_media_hash": null,
  "edit_history": []
}
```

Clients:
- **MUST** treat missing provenance or `origin_type = "UNKNOWN"` as “no strong provenance evidence,” not as an error.
- **SHOULD NOT** infer provenance from omission; explicit `UNKNOWN` keeps the default machine-readable.

### 5.1 Media Storage & Availability

- Media referenced by `media_hash` **MUST** be content‑addressed (e.g., IPFS/Arweave hash).
- Responsibility for persistence:
  - Authors **SHOULD** pin or otherwise ensure availability of referenced media.
  - Relays **MAY** cache/pin but are not required to store media blobs.
  - Bootstrap documents may list recommended pinning services; clients **SHOULD** warn when media is unpinned/unavailable.
- If media cannot be fetched or its hash mismatches, clients **MUST** downgrade provenance to `UNKNOWN` for that media reference.

## 6. Genesis Events
A Genesis Event is a standalone object that records the initial origin of specific media.

### 6.1 Genesis Event Structure

```jsonc
{
  "genesis_id": "0x1e20b1c2d3e4f5061728394a5b6c7d8e9f0a1b2c3d4e5f6071829304a5b6c7d8e9",
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
  - Structured evidence binding device/model/app to hardware/platform. See §6.3 for supported formats.

Genesis Events are referenced by genesis_media_hash or genesis_id from Packets.
They allow independent verification of hardware/model claims.

### 6.2 Software Details and Capture Methods

For `origin_type = SOFTWARE`, the `details.software` object supports an extended schema for app‑based capture:

```jsonc
{
  "software": {
    "software_id": "did:strata:official_camera_app",
    "software_version": "1.0.0",
    "creation_timestamp": 1715421000,
    "capture_method": "DIRECT_CAMERA_API",
    "device_metadata": {
      "platform": "ios",
      "os_version": "17.0",
      "device_model": "iPhone 15 Pro",
      "gps": { "lat": 37.7749, "lon": -122.4194 }
    }
  }
}
```

Fields:
- `software_id` – StrataID (DID) of the capture application. Clients use this to determine app trust level.
- `software_version` – Version string of the software.
- `creation_timestamp` – Unix epoch seconds when capture/creation occurred.
- `capture_method` (optional) – How media was acquired:
  - `DIRECT_CAMERA_API` – Captured directly from device camera API (not from gallery/file system).
  - `SCREEN_CAPTURE` – Captured from screen recording.
  - `FILE_IMPORT` – Imported from file system or gallery.
  - `GENERATED` – Programmatically generated (non‑AI).
  - Omitted or `null` – Unknown capture method.
- `device_metadata` (optional) – Best‑effort device information (not cryptographically verified):
  - `platform` – `ios`, `android`, `web`, `desktop`.
  - `os_version` – Operating system version string.
  - `device_model` – Device model identifier.
  - `gps` – Location at capture time (if permitted by user).

**Trust Hierarchy for SOFTWARE origin:**

| Scenario | Trust Level | Traffic Light |
|----------|-------------|---------------|
| Trusted app + platform attestation + DIRECT_CAMERA_API | High | Yellow‑Green |
| Trusted app + DIRECT_CAMERA_API (no platform attestation) | Medium | Yellow |
| Trusted app + FILE_IMPORT | Low | Yellow |
| Unknown app or no capture_method | Minimal | Yellow |

Clients **MUST** verify `software_id` against their trusted capture apps list (see RFC‑0006) before elevating trust.

### 6.3 Attestation Evidence Formats

The `attestation_evidence` field supports multiple formats for binding genesis claims to verifiable platform assertions:

```jsonc
{
  "attestation_evidence": {
    "format": "<format-identifier>",
    "payload": "base64url(...)",
    "nonce": "base64url(...)"
  }
}
```

**Supported formats:**

| Format | Platform | What It Proves |
|--------|----------|----------------|
| `android-key-attestation` | Android | Key generated in hardware TEE; device not rooted |
| `apple-device-attestation` | iOS | Device has secure enclave; capture signed by device |
| `apple-app-attest` | iOS | App is genuine (correct bundle ID), running on real device, unmodified |
| `play-integrity` | Android | App is genuine (correct package), device passes integrity checks |
| `c2pa` | Cross‑platform | C2PA manifest with claim/assertion chain |

**Format: `apple-app-attest`**

Used by Strata capture apps on iOS to prove app authenticity without hardware‑level capture signing.

```jsonc
{
  "format": "apple-app-attest",
  "payload": "base64url(CBOR attestation object from DCAppAttestService)",
  "nonce": "base64url(SHA256(genesis_id || media_hash))"
}
```

Verification:
1. Decode CBOR attestation object from `payload`.
2. Verify attestation certificate chain roots to Apple App Attest CA.
3. Verify `nonce` matches `SHA256(genesis_id || media_hash)`.
4. Extract `rpIdHash` and verify it matches expected app bundle ID hash.
5. Verify counter and risk metric if present.

**Format: `play-integrity`**

Used by Strata capture apps on Android to prove app and device integrity.

```jsonc
{
  "format": "play-integrity",
  "payload": "base64url(integrity token JWT)",
  "nonce": "base64url(SHA256(genesis_id || media_hash))"
}
```

Verification:
1. Decode and verify JWT signature against Google's public keys.
2. Verify `nonce` claim matches `SHA256(genesis_id || media_hash)`.
3. Check `appIntegrity.appRecognitionVerdict` is `PLAY_RECOGNIZED`.
4. Check `deviceIntegrity.deviceRecognitionVerdict` meets minimum threshold.
5. Verify `requestDetails.requestPackageName` matches expected package.

**Format: `c2pa`**

Used when media includes a C2PA manifest (e.g., from Adobe tools or hardware with C2PA support).

```jsonc
{
  "format": "c2pa",
  "payload": "base64url(JUMBF C2PA manifest)",
  "nonce": null
}
```

Verification:
1. Parse JUMBF manifest and extract claim/assertion chain.
2. Verify signatures against C2PA trust list or known roots.
3. Map C2PA assertions to Strata provenance claims.

### 6.4 Verification Requirements

- For `origin_type = HARDWARE_SECURE_ENCLAVE`, Genesis Events **MUST** include `issuer_chain` that chains to hardware manufacturer roots distributed in bootstrap documents; clients **MUST** validate the chain and attestation evidence. If missing or invalid, provenance for that media **MUST** be treated as `UNKNOWN/SOFTWARE`.
- For `origin_type = AI_MODEL`, `issuer_chain` anchors the model/provider key; clients **MUST** verify the chain against model provider roots from bootstrap documents or local policy.
- For `origin_type = SOFTWARE` with `capture_method = DIRECT_CAMERA_API`:
  - If `attestation_evidence` with `apple-app-attest` or `play-integrity` is present and valid, AND `software_id` is in the client's trusted capture apps list, clients **MAY** treat this as elevated trust (yellow‑green).
  - If attestation is missing or invalid, clients **MUST** treat as standard SOFTWARE (yellow).
- `device_public_key` and model keys are authenticated via the validated chain; unsigned or self‑asserted keys are insufficient for green‑ring treatment.
- App attestation (`apple-app-attest`, `play-integrity`) does NOT prove hardware capture; it proves the app is genuine. Clients **MUST NOT** treat app attestation as equivalent to hardware attestation.

## 7. Consensus Metrics

The in‑packet `consensus_metrics` field is **RESERVED**. Authors **MUST NOT** populate it; clients **MUST** ignore unsigned values.

Aggregated counters **MUST** be published as standalone Packets with `content.type = "CONSENSUS_METRIC"`:

```jsonc
{
  "content": {
    "type": "CONSENSUS_METRIC",
    "target_packet": "0x1e208f2c3a4b5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f7a1",
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
