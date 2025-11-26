
# RFC-0004: Relay Transport Protocol

- **Status:** Draft
- **Author(s):** Strata Core Team
- **Created:** 2025-11-25
- **Updated:** 2025-11-26
- **Scope:** Normative protocol (reference relay transport)

> **Note:** `packet_id` uses `0x` + lowercase hex encoding. See RFC-0000 5.2-5.4.  

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
    "since": 1715420000,
    "until": 1715429999,
    "limit": 50,
    "cursor": null,
    "order": "asc"                 // asc|desc for historical catchup
  }
}
```

Fields:
- `subscription_id`: client‑assigned ID to correlate responses.
- `filters`: optional filter criteria:
  - `authors`: list of author_ids,
  - `types`: list of content.types,
  - `since`: Unix timestamp; only return Packets with timestamp >= since.
  - `until`: Unix timestamp upper bound for historical fetch.
  - `limit`: maximum historical Packets to return immediately (relay MAY cap).
  - `cursor`: opaque paging token returned by the relay in previous `EVENT`s.
  - `order`: `asc` or `desc` for historical catchup; default `asc`.

Relay behavior:
- **MAY** send up to `limit` historical Packets matching filters, ordered by `received_at`.
- **MUST** send new matching Packets as `EVENT` messages.
- **MUST** include a `cursor` in historical `EVENT`s to allow pagination; clients MAY re‑SUBSCRIBE with that `cursor` to continue.

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
  "packet": { /* full Packet JSON */ },
  "received_at": 1715423200,   // relay-observed timestamp
  "cursor": "opaque-token-1"   // for pagination
}
```

Relays:
- **MUST** include `subscription_id` if the Packet is delivered due to a `SUBSCRIBE`.
- **MAY** send unsolicited `EVENT`s if client has a “default feed” subscription.
- **MUST** include `received_at` so clients can enforce freshness and key validity (see RFC‑0002).

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
- `unauthenticated`
- `internal_error`

### 3.6 UNSUBSCRIBE

Client → Relay:

```json
{
  "type": "UNSUBSCRIBE",
  "subscription_id": "sub-123"
}
```

Relay behavior:
- **MUST** stop sending `EVENT`s for the given `subscription_id`.
- **SHOULD** free any server‑side resources associated with the subscription.

### 3.7 AUTH / AUTH_CHALLENGE / AUTH_OK

Relay → Client (optional but RECOMMENDED):

```json
{
  "type": "AUTH_CHALLENGE",
  "challenge": "base64random...",
  "server_id": "did:strata:relay_1",
  "expires_at": 1715422000
}
```

Client → Relay:

```json
{
  "type": "AUTH",
  "author_id": "did:strata:alice",
  "challenge": "base64random...",
  "signature": "0xSigOver(challenge || server_id)",
  "capabilities": ["publish", "subscribe"],
  "client_msg_id": "optional-id"
}
```

Relay → Client (on success):

```json
{
  "type": "AUTH_OK",
  "client_msg_id": "optional-id",
  "author_id": "did:strata:alice",
  "capabilities": ["publish", "subscribe"]
}
```

Rules:
- Relays **SHOULD** issue `AUTH_CHALLENGE` on connect; clients that cannot authenticate remain anonymous.
- `AUTH` **MUST** be signed with an active key for `author_id`.
- After receiving an `AUTH` message, the relay **MUST** respond promptly with either:
  - `AUTH_OK` if authentication and capability checks succeed, or
  - `ERROR` with `code` set to `unauthenticated`, `invalid_signature`, or a more specific auth‑related error if the request is rejected.
- If the relay enforces a capability model, the effective capability set for the session **MUST** be returned in `AUTH_OK.capabilities`. Clients **SHOULD** treat this array as authoritative for what they may attempt on that connection.
- Clients **SHOULD** reuse the authenticated WebSocket for all actions instead of re‑authenticating per message.
- Challenge validity: `expires_at` **MUST** allow at least 60 seconds of validity; recommended default is 120 seconds and **SHOULD NOT** exceed 10 minutes. Relays **MUST** reject shorter windows, and clients **SHOULD** retry with a fresh challenge if one expires mid‑handshake (use small randomized backoff to avoid thundering herds).

Client behavior:
- Start a short timeout after sending `AUTH` (e.g., a few seconds).
- Transition the connection to `authenticated` only upon receiving `AUTH_OK`.
- Treat `ERROR code: unauthenticated` or `ERROR code: invalid_signature` in response to `AUTH` as an authentication failure and return to `unauthenticated`.
- On timeout with no `AUTH_OK` or `ERROR`, drop back to `unauthenticated` and **MAY** retry a fresh `AUTH` or a new connection.

Correlation IDs:
- Clients **MAY** include a `client_msg_id` field in any request‑like message that expects a direct reply, including `PUBLISH` and `AUTH`.
- Relays **SHOULD** echo `client_msg_id` in `OK`, `ERROR`, and `AUTH_OK` responses so clients can correlate replies with requests.
- Relays **MUST** tolerate clients that omit `client_msg_id`.

Clients **MUST NOT** assume success based solely on the absence of an `ERROR`; they **SHOULD** wait for `AUTH_OK` or an explicit `ERROR` until their auth timeout elapses.

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
- Record `received_at` (relay‑observed time) for each stored Packet and use it for ordering.
- Support cursor‑based pagination over historical queries ordered by `received_at`.

## 5. Encryption

### 5.1 Public vs Private Content

- Public posts:
  - `content` MAY be plaintext.
  - Provenance, attestations, and metadata are visible.
- Private messages (`content.type = "MESSAGE"`):
  - `content` MUST be end‑to‑end encrypted.
  - Recommended baseline:
    - Key agreement: X25519,
    - AEAD: XChaCha20‑Poly1305,
    - KDF: HKDF‑SHA256,
    - Forward secrecy: Double Ratchet or MLS session per conversation.
  - Envelope shape (per Packet):
```jsonc
{
  "type": "MESSAGE",
  "encryption": {
    "scheme": "x25519-xchacha20poly1305-dratchet-v1",
    "recipients": [
      {
        "id": "did:strata:bob",
        "header": "base64encoded_dh_and_ratchet_state",
        "ciphertext": "base64ciphertext"
      }
    ]
  }
}
```
  - Relays should see only:
    - Encrypted blob,
    - Minimal routing metadata (recipient DIDs or blinded tokens if possible).

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

### 6.1 Rate Limiting Semantics

- When a request is rate limited, relays **MUST** respond with `ERROR code: rate_limited` and include:
  - `retry_after` (seconds),
  - `limit` (requests allowed per window, optional),
  - `remaining` (optional),
  - `window_seconds` (optional).
- Rate limits MAY be scoped to authenticated identity (preferred) or IP address (fallback).
- Clients **SHOULD** distinguish `rate_limited` from other failures and back off accordingly.

## 7. Client-Side Relay QoS Reputation

This is distinct from user/identity reputation; relays are untrusted utilities evaluated on performance/integrity.

- Local scope: Each client keeps a local `relay_scores` table; there is no global consensus score.
- Passive metrics:
  - Latency (round-trip/ping),
  - Throughput for media blobs,
  - Uptime/connection success rate over a rolling 7‑day window.
- Active integrity:
  - Validity check: drop/penalize relays that forward malformed or tampered Packets (signature mismatch with content).
  - Consistency check (anti-censorship/silent-drop): randomized probes for known packet hashes; penalize relays returning 404 for widely available content.
- State machine (example):
  - 🟢 Active: preferred for fetch/publish.
  - 🟡 Backup: slow but functional; used when Active relays fail.
  - 🔴 Blacklisted: repeated integrity failures/timeouts; stop connecting.
- Churn: Clients SHOULD periodically sample new relays from bootstrap lists to avoid sticking to degraded sets.

## 8. Security Considerations

Traffic correlation:
- Relays can observe who connects when and how often; metadata minimization or layered routing may be necessary for high‑risk users.
Denial‑of‑Service:
- Relays SHOULD implement rate limits and basic DoS protections.
Abuse:
- Relays may be used to distribute harmful content; operators need legal strategies.
- Authentication:
  - `AUTH` is optional but recommended; unauthenticated connections should be sandboxed and more tightly rate‑limited.
- Replay:
  - Relays **MUST** drop duplicate `packet_id`s and honor `expires_at`/freshness from RFC‑0002 to prevent replay amplification.

## 9. Backwards Compatibility

Future versions of this transport protocol may add:
- New message types,
- Additional filter fields,
- Alternative transports (HTTP/2, QUIC).

Relays SHOULD ignore unrecognized fields and treat unknown `type` messages as errors without closing the connection when possible.
