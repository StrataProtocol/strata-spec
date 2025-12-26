# RFC-0101: Strata over ActivityPub (Transport Binding)

- **Status:** Draft  
- **Scope:** Transport binding (normative once accepted)  
- **Author(s):** Strata Core Team  
- **Created:** 2025-12-13  
- **Updated:** 2025-12-13  

## 1. Summary

This RFC defines how Strata data can be carried across an ActivityPub federation by embedding **canonical Strata objects** as ActivityPub extensions:

- A Strata **Packet** (RFC‑0002) MAY be embedded inside an ActivityPub Activity/Object via a `strata:packet` extension.
- A Strata **provenance_header** (RFC‑0002) and **attestations** (RFC‑0003) MAY be embedded as ActivityPub extensions, either as part of the embedded Packet or (in “metadata-only” mode) as standalone extension fields.
- Nodes that also implement a native Strata relay **MUST** treat inbound ActivityPub-derived Packets as untrusted input and apply the same validation and dedup rules required by RFC‑0002/0004/0007.

This binding enables three deployment profiles:

1. **Metadata-only**: Attach provenance/attestation metadata to normal ActivityPub content (no Strata Packet required).
2. **Packet-in-ActivityPub**: Carry full signed Strata Packets over ActivityPub federation.
3. **Bridge**: A dual-stack node that speaks ActivityPub to the Fediverse and RFC‑0004/0007 to Strata relays.

## 2. Goals and Non-goals

### 2.1 Goals

- Allow Strata-aware clients and relays to ingest/verifiably store Strata Packets transported over ActivityPub.
- Minimize coupling: ActivityPub implementations that ignore unknown fields remain compatible.
- Preserve Strata’s security invariants: packet immutability, signature verification, dedup by `packet_id`, and relay witness time (`received_at`).

### 2.2 Non-goals

- Define a complete ActivityPub server implementation.
- Define end-to-end encryption for ActivityPub messaging (out of scope in Strata core; see RFC‑0004 §5 note).
- Replace native Strata relay transport (RFC‑0004) or relay federation (RFC‑0007).

## 3. Terminology

- **AP**: ActivityPub.
- **AP Actor**: An ActivityPub `Actor` (e.g., `Person`, `Service`).
- **AP Activity**: An ActivityStreams `Activity` (e.g., `Create`, `Announce`, `Like`, `Follow`).
- **AP Object**: The `object` of an Activity (e.g., `Note`, `Article`).
- **Strata-aware AP node**: An ActivityPub node that understands this RFC’s extensions.
- **Bridge**: A node that both (a) federates via ActivityPub and (b) implements a Strata relay interface (RFC‑0004), optionally with relay↔relay replication (RFC‑0007).

Normative language follows RFC‑0000.

## 4. JSON-LD Context and Namespace

Strata-aware AP nodes SHOULD include a JSON-LD context entry defining the `strata` prefix:

```jsonc
"@context": [
  "https://www.w3.org/ns/activitystreams",
  "https://w3id.org/security/v1",
  { "strata": "https://strataprotocol.org/ns#" }
]
```

Notes:
- The IRI is an identifier for JSON-LD compact IRIs; it does not need to resolve to be useful.
- Strata payloads embedded under Strata extension keys are **plain JSON** using Strata conventions (RFC‑0000 `snake_case`), and MUST NOT be JSON-LD expanded or rewritten.

## 5. Extension Fields

### 5.1 `strata:packet` (Packet-in-ActivityPub profile)

When transporting a Strata Packet over ActivityPub, a Strata-aware node MUST include a `strata:packet` field containing the **full Strata Packet JSON** (RFC‑0002), either:

- As a JSON object, or
- As a UTF‑8 JSON string whose parsed value is a JSON object.

The embedded Packet:
- MUST be valid per RFC‑0002 (including canonical `packet_id` and `signature` rules).
- MUST use Strata’s `snake_case` field names (RFC‑0000 §4.3).
- MUST NOT be modified by intermediate nodes.

Placement:
- `strata:packet` SHOULD appear on the **AP Object** (the `object` of an Activity).
- `strata:packet` MAY appear on the AP Activity itself, but if both are present they MUST be byte-equivalent after JSON parsing.

### 5.2 `strata:packet_id` (Optional)

Nodes MAY include `strata:packet_id` (string) as a redundant index hint. If present:
- It MUST equal the embedded Packet’s `packet_id` (RFC‑0002).
- Receivers MUST NOT trust it without validating the Packet.

### 5.3 `strata:provenance_header` and `strata:attestations` (Metadata-only profile)

Nodes MAY attach Strata provenance/attestation data to normal ActivityPub Objects without embedding a full Packet:

- `strata:provenance_header`: a provenance header object per RFC‑0002 §5.
- `strata:attestations`: an array of attestation objects per RFC‑0003 §3.

If both `strata:packet` and metadata-only fields are present, the embedded Packet is authoritative and receivers MUST ignore conflicting metadata-only fields.

## 6. Receiver Validation Rules (Packet-in-ActivityPub)

A receiver that supports Strata Packets over ActivityPub MUST:

1. Extract the embedded Packet from `strata:packet`.
2. Validate Packet fields and compute `packet_id` per RFC‑0002 §4.2 (canonicalization + BLAKE3‑256).
3. Verify the Packet `signature` using the author’s public key (RFC‑0001).
4. Apply freshness and expiry rules:
   - Reject Packets too far in the future or too old per RFC‑0002 §3.3 (unless explicitly configured for historical backfill).
   - If `expires_at` is present and in the past, the Packet MUST be ignored (RFC‑0002 §3.3).
5. Deduplicate by `packet_id` (RFC‑0002 §3.3, RFC‑0007 §7).

If any validation step fails, the receiver MUST ignore the embedded Packet. It MAY still process the ActivityPub Activity as a normal AP object (depending on local policy).

Transport authentication:
- ActivityPub HTTP Signatures are useful for ActivityPub-layer authenticity, but do not replace Strata Packet signature validation. Strata-aware receivers MUST treat the Packet as untrusted until the Packet signature and `packet_id` validate.

## 7. Identity Binding Between ActivityPub and Strata

### 7.1 `strata:id` on Actors (Optional)

An ActivityPub Actor MAY publish a StrataID on its Actor object:

```jsonc
{
  "type": "Person",
  "id": "https://example.social/users/alice",
  "preferredUsername": "alice",
  "strata:id": "did:strata:z6Mk..."
}
```

This is advisory metadata. The authoritative binding for Strata is always the Packet’s `author_id` and signature verification (RFC‑0002/0001).

### 7.2 Self-certifying StrataIDs (RECOMMENDED for bridges)

For interoperability with relays that do not have access to an external DID resolution service, bridges SHOULD use **self-certifying StrataIDs** where the method-specific ID encodes the Ed25519 public key as multibase `z` (base58btc), optionally with the Ed25519 multicodec prefix `0xed01`.

This allows receivers to derive a public key directly from `author_id` for signature verification without fetching a DID document.

### 7.3 Bridged identities (AP-only users)

A bridge MAY synthesize a Strata identity for an AP Actor in order to create signed Strata Packets representing ActivityPub content.

Requirements:
- The mapping from AP Actor URI → StrataID MUST be stable across restarts.
- The bridge MUST be able to sign Packets for that StrataID.
- The bridge SHOULD store the Actor URI ↔ StrataID mapping locally and MAY publish it as non-authoritative metadata (e.g., in its own database, or via a signed CONFIG packet in the future).

Security note:
- These synthesized identities represent the bridge’s view of the Fediverse and SHOULD be labeled as such in clients (e.g., via `app_metadata.extras`), since they are not keys controlled by the AP user.

## 8. Recommended Activity Mapping (Bridge Profile)

This section is a RECOMMENDED interoperability profile for bridges that translate ActivityPub social actions into RFC‑0200 social content types so that Strata-native clients can render fediverse content with a consistent schema.

### 8.1 ActivityPub → Strata (RECOMMENDED)

- `Create(Note)` → `content.type = "POST"` with `post_kind = "ORIGINAL"`.
- `Create(Article)` → `content.type = "POST"` with `post_kind = "ARTICLE"`.
- `Announce` → `content.type = "POST"` with `post_kind = "REPOST"`.
- `Like` → `content.type = "REACTION"` with `reaction_kind = "LIKE"`.
- `Follow` → `content.type = "SOCIAL_EDGE"` with `relation = "FOLLOW"`.
- `Undo(Follow)` → `content.type = "SOCIAL_EDGE_REVOCATION"` (RFC‑0200 §7.3).
- `Delete` (of an object) → `content.type = "CORRECTION"` with `action = "retract"` targeting the referenced Packet.
- `Update` (of a Note/Article) → `content.type = "CORRECTION"` with `action = "replace"` when a target Packet can be identified.

Target linking:
- If the bridged ActivityPub object includes `strata:packet` or `strata:packet_id`, the bridge SHOULD use that `packet_id` for Strata references (`thread_ref`, `repost_of`, `target_packet`, etc.).
- If a Strata `packet_id` cannot be determined, the bridge MUST NOT emit a nonconformant Strata social content object:
  - For mappings where the Strata schema requires a `packet_id` reference (e.g., `POST`/`REPLY.thread_ref`, `POST`/`REPOST.repost_of`, `REACTION.target_packet`, `CORRECTION.target_id`), the bridge SHOULD either:
    - Not emit a Strata Packet for that ActivityPub activity, or
    - Emit a `POST`/`ORIGINAL` describing the activity and record the ActivityPub target IRI under `app_metadata.extras.activitypub.target_uri`.
  - For optional references, the bridge MAY omit the reference field.
  - Bridges MAY record the ActivityPub IRI(s) in bridge-specific metadata while keeping Strata reference fields unset.

### 8.2 Strata → ActivityPub (RECOMMENDED)

Bridges translating Strata packets into ActivityPub activities for fediverse visibility SHOULD:

- POST/ORIGINAL, POST/REPLY → `Create(Note)` (or `Create(Article)` for POST/ARTICLE).
- POST/REPOST → `Announce`.
- REACTION/LIKE → `Like`.
- SOCIAL_EDGE/FOLLOW → `Follow`.
- SOCIAL_EDGE_REVOCATION (FOLLOW) → `Undo(Follow)`.
- CORRECTION retract → `Delete`.
- CORRECTION replace → `Update`.

In all cases, the ActivityPub payload SHOULD include `strata:packet` so Strata-aware nodes can validate the authoritative Strata Packet, even if the ActivityPub fields are used for compatibility with non-Strata AP clients.

### 8.3 Bridge metadata keys (RECOMMENDED)

Bridges MAY include an `app_metadata.extras.activitypub` object (RFC‑0200 `app_metadata`) to preserve ActivityPub identifiers without violating Strata schemas:

- `actor_uri`: ActivityPub actor IRI (string)
- `activity_uri`: ActivityPub activity IRI (string)
- `object_uri`: ActivityPub object IRI (string)
- `target_uri`: ActivityPub target IRI when Strata `packet_id` reference is unknown (string)
- `ap_type`: original ActivityPub `type` (string)

## 9. Media and Provenance Guidance

When translating ActivityPub media attachments into Strata `media` entries (RFC‑0002/RFC‑0200):

- `media_hash` MUST be content-addressed to the media bytes (e.g., `0x` + hex(BLAKE3‑256(media_bytes))).
- Bridges SHOULD populate `media_uris` (RFC‑0002 `media` entries) with any fetchable attachment URLs/locators (e.g., the ActivityPub attachment `url`) as retrieval hints, but receivers MUST still verify fetched bytes against `media_hash`.
- If a bridge cannot fetch media bytes, it SHOULD omit the media entry rather than emit a non-content-addressed placeholder.
- Packets that include media MUST include a `provenance_header` (RFC‑0002). For ActivityPub-origin media with unknown provenance, bridges SHOULD emit the minimum unknown header (RFC‑0002 §5.0):

```jsonc
{
  "origin_type": "UNKNOWN",
  "genesis_media_hash": null,
  "edit_history": []
}
```

## 10. Interaction with Strata Relays and Relay Federation

Bridges that implement a native Strata relay interface:

- MUST implement RFC‑0004 for Strata client/relay interop.
- MUST apply the same validation pipeline to Packets received via ActivityPub as to Packets received via `PUBLISH` (RFC‑0007 §4.1).
- MUST deduplicate by `packet_id` and MUST NOT mutate Packet bodies.
- SHOULD preserve and store the earliest known witness time for a Packet when learning it from multiple sources, consistent with RFC‑0002 §3.3 and RFC‑0007 §6.3.

Loop avoidance:
- Bridges SHOULD keep a short-lived cache of recently emitted `packet_id`s per output transport (ActivityPub vs Strata relay mesh) and MUST NOT re-emit the same Packet onto the same transport due to re-ingest from that transport.

## 11. Security and Privacy Considerations

- **Untrusted content**: ActivityPub is adversarial; Strata-aware receivers MUST validate embedded Packets and MUST ignore invalid objects.
- **Key trust**: If a bridge synthesizes identities, clients SHOULD label such content as “bridged” and treat it with appropriate trust defaults.
- **Replay and storms**: Implementations SHOULD enforce per-peer rate limits and dedup caches; relays/bridges SHOULD cap historical backfill and validate cursors if supported (RFC‑0004/0007).
- **PII leakage**: Bridges SHOULD avoid embedding personally identifying metadata in Packets; if AP content contains PII, it is already public in the AP network and should be treated accordingly.

## 12. Open Questions (Draft)

- Standardized way to publish bridge identity mappings (AP Actor ↔ StrataID) as signed Strata config packets.
- Canonical namespace IRI for `strata:` JSON-LD context.
- Whether to define a dedicated ActivityStreams object type for “StrataPacket” vs always using Note/Article for maximum compatibility.
