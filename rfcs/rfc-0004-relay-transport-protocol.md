
# RFC-0004: Relay Transport Protocol

- **Status:** Draft  
- **Author(s):** Strata Core Team  
- **Created:** 2025-11-25  
- **Updated:** 2025-11-25  

---

## 1. Summary

This RFC defines a reference **transport protocol** for Strata relays:

- Relay responsibilities and non‑responsibilities,
- A minimal WebSocket‑based API for publish/subscribe,
- Guidelines for encryption and opaque handling of content.

Implementations MAY use different transport mechanisms, but SHOULD offer a WebSocket endpoint following this spec for interoperability.

---

## 2. Relay Responsibilities

Relays:

- **MUST** accept valid Packets from clients.
- **MUST NOT** modify Packet contents.
- **MAY**:
  - Reject Packets based on local policy (e.g. invalid signatures, rate limits),
  - Decline to store or forward specific `packet_id`s or media hashes,
  - Block specific `author_id`s.

Relays are **not** responsible for:

- Fact‑checking content,
- Determining truthfulness,
- Assembling user feeds.

Relays SHOULD treat packet payloads as opaque data.

## 3. WebSocket Message Types

Relays expose a WebSocket endpoint, e.g.:

```text
wss://relay.example.org/strata
```
### 3.1 SUBSCRIBE

Client → Relay:

```json
{
  "type": "SUBSCRIBE",
  "subscription_id": "sub-123",
  "filters": {
    "authors": ["did:strata:zero_cool"],
    "types": ["POST", "ATTESTATION"],
    "since": 1715420000
  }
}
```

Fields:
- `subscription_id`: client‑assigned ID to correlate responses.
- `filters`: optional filter criteria:
  - `authors`: list of author_ids,
  - `types`: list of content.types,
  - `since`: Unix timestamp; only return Packets with timestamp >= since.

Relay behavior:
- **MAY** send historical Packets matching filters.
- **MUST** send new matching Packets as `EVENT` messages.

### 3.2 PUBLISH

Client → Relay:

```json
{
  "type": "PUBLISH",
  "packet": { /* full Packet JSON */ }
}
```

Relay behavior:
- **MUST** verify packet signature and schema,
- **MAY** check against local policy,
- **On success**:
  - Stores Packet,
  - Forwards to relevant subscribers,
  - Sends `OK` response.

### 3.3 EVENT

Relay → Client:

```json
{
  "type": "EVENT",
  "subscription_id": "sub-123",
  "packet": { /* full Packet JSON */ }
}
```

Relays:
- **MUST** include `subscription_id` if the Packet is delivered due to a `SUBSCRIBE`.
- **MAY** send unsolicited `EVENT`s if client has a “default feed” subscription.

### 3.4 OK

Relay → Client:

```json
{
  "type": "OK",
  "client_msg_id": "optional-id",
  "packet_id": "0x8f2...a1",
  "status": "ACCEPTED"    // or "QUEUED"
}
```
Clients MAY include a `client_msg_id` field in their `PUBLISH` messages; relays SHOULD echo it in `OK`/`ERROR`.

### 3.5 ERROR

Relay → Client:

```json
{
  "type": "ERROR",
  "client_msg_id": "optional-id",
  "code": "invalid_signature",
  "reason": "Signature verification failed"
}
```
Common codes:
- `invalid_signature`
- `invalid_schema`
- `rate_limited`
- `blocked_author`
- `internal_error`

## 4. Storage & Indexing

This RFC does not mandate a storage backend. A relay MAY use:
- In‑memory store (for lightweight relays),
- Key‑value store (e.g., RocksDB),
- Relational DB (e.g., PostgreSQL),
- Content‑addressed stores (for large media).

However, relays MUST:
- Be able to retrieve Packets by `packet_id`.
- Be able to query Packets by basic filters:
  - `author_id`,
  - `content.type`,
  - `timestamp range`.

## 5. Encryption

### 5.1 Public vs Private Content

- Public posts:
  - `content` MAY be plaintext.
  - Provenance, attestations, and metadata are visible.
- Private messages (`content.type = "MESSAGE"`):
  - `content` SHOULD be end‑to‑end encrypted.
  - Relays should see only:
    - Encrypted blob,
    - Metadata necessary for routing (e.g. recipients’ DIDs, possibly encrypted).

### 5.2 Opaque Payload Handling

Relays:
- **MUST NOT** attempt to decrypt encrypted fields.
- **SHOULD NOT** inspect or log decrypted content even if misconfigured.

Relays MAY log:
- Packet metadata (IDs, timestamps, author IDs),
- Generic metrics (counts, sizes),
- Policy decisions (e.g., blocked, accepted).

## 6. Relay Policies

While the protocol is neutral, relays may:
- Decline to store or forward content that is illegal in their jurisdiction,
- Block specific identities for abuse or spam,
- Limit resource usage.

It is recommended that:
- Relay policies be documented and public,
- Relay operators publish basic policy summaries (e.g., in a manifest).

## 7. Security Considerations

Traffic correlation:
- Relays can observe who connects when and how often; metadata minimization or layered routing may be necessary for high‑risk users.
Denial‑of‑Service:
- Relays SHOULD implement rate limits and basic DoS protections.
Abuse:
- Relays may be used to distribute harmful content; operators need legal strategies.

## 8. Backwards Compatibility

Future versions of this transport protocol may add:
- New message types,
- Additional filter fields,
- Alternative transports (HTTP/2, QUIC).

Relays SHOULD ignore unrecognized fields and treat unknown `type` messages as errors without closing the connection when possible.
