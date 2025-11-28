<!-- rfcs/rfc-0007-relay-federation-and-replication.md -->

# RFC-0007: Relay Federation & Replication

- **Status:** Draft  
- **Author(s):** Strata Core Team  
- **Created:** 2025-11-27  
- **Updated:** 2025-11-28  
- **Scope:** Normative protocol (relay–relay behavior on top of RFC‑0004)

## 1. Summary

Defines how Strata relays federate and replicate data between each other using the existing WebSocket transport (RFC‑0004). Covers:
- Relay authentication,
- Replication sessions and subscription profiles,
- Optional replication policy publishing,
- Safeguards against loops, replay storms, and abuse.

No new Packet types are introduced; this RFC reuses RFC‑0000/0002 Packet format and RFC‑0004 transport.

## 2. Motivation

Relays form the transport mesh (Layer 2). Federation enables:
- Redundancy and locality for users,
- Regional mirrors and archival relays,
- Global-ish visibility for apps without central servers.

Standardized replication lets operators build meshes and bridges predictably and lets clients understand relay connectivity.

## 3. Terminology

- **Relay** – per RFC‑0000/0004: node that accepts, stores, forwards Packets.
- **Federated relay** – replicates with at least one other relay.
- **Peer relay** – another relay in a replication relationship.
- **Replication session** – WebSocket connection where one relay subscribes to another.
- **Upstream / downstream** – for a session: upstream is the relay being subscribed to; downstream receives replicated Packets.

## 4. Preconditions

### 4.1 Packet & transport compliance
Federating relays:
- **MUST** implement RFC‑0004 transport.
- **MUST** implement validation from RFC‑0000/0002 (canonicalization, `packet_id`, signature verification, freshness, `expires_at`, dedup by `packet_id`).
- **MUST NOT** mutate replicated Packets.
- **MUST** apply the same validation pipeline to Packets received via replication as to user-submitted Packets; invalid Packets **MUST NOT** be stored or served.

### 4.2 Relay identities
Each federating relay **MUST** have a StrataID (`did:strata:...`, RFC‑0001) with a DID Document suitable for signing:
- `AUTH` messages, and
- Optional `CONFIG` Packets for replication policy.

Relays **SHOULD** advertise their relay DID in bootstrap documents and HTTP manifests (see §10).

## 5. Authentication between relays

### 5.1 Using AUTH
Relays **MUST** authenticate replication sessions with `AUTH_CHALLENGE` / `AUTH` / `AUTH_OK` from RFC‑0004.
- Upstream **SHOULD** send `AUTH_CHALLENGE` on connect.
- Downstream **MUST** respond with `AUTH`:
  - `author_id` = its relay DID,
  - `signature` proves key possession,
  - `capabilities` **MUST** include `"replicate"` if replication is intended.

```jsonc
{
  "type": "AUTH",
  "author_id": "did:strata:relay_eu1",
  "challenge": "base64random...",
  "signature": "0xSig...",
  "capabilities": ["subscribe", "replicate"]
}
```

Upstream **MUST** answer with `AUTH_OK` or `ERROR` (`code: "unauthenticated"`). Relays **MAY** refuse replication from peers lacking `"replicate"`.

### 5.2 Capability model
Relays MAY implement capability vocabularies; this RFC only requires recognizing `"replicate"`. Typical meanings:
- `"publish"` – permission to `PUBLISH`,
- `"subscribe"` – permission to issue general `SUBSCRIBE`,
- `"replicate"` – identifies infrastructure peers and MAY grant higher limits/backfill.
- Upstreams **MUST** combine capability checks with local allow-lists/policy before granting replication; unknown capability strings **MUST** be ignored.

## 6. Replication subscriptions

### 6.1 SUBSCRIBE for replication
A replication subscription is any `SUBSCRIBE` where the authenticated identity has `"replicate"` and filters target replication.

Example (full-archive):

```jsonc
{
  "type": "SUBSCRIBE",
  "subscription_id": "replica-eu1",
  "filters": {
    "authors": [],
    "types": [],
    "since": 1715400000,
    "until": null,
    "limit": 500,
    "cursor": null,
    "order": "asc"
  },
  "replication_profile": "FULL_ARCHIVE"
}
```

Rules:
- `authors = []` and `types = []` mean “no restriction”.
- `replication_profile` is an extension hint; upstreams MAY ignore it. Unknown fields MUST be ignored per RFC‑0000.

### 6.2 Recommended profiles
Implementations **SHOULD** support these profile names:
- `FULL_ARCHIVE` – all content types, up to upstream’s max backfill.
- `PUBLIC_SOCIAL` – public social types (e.g., POST, PROFILE, SOCIAL_EDGE, REACTION, GROUP, GROUP_MEMBERSHIP, CONSENSUS_METRIC, ATTESTATION; future RFC‑0200 refines).
- `HIGH_TRUST_ONLY` – only Packets from authors meeting upstream’s trust threshold (RFC‑0005); non‑deterministic per relay.

Profiles are advisory; relays MAY map/reject with `ERROR` if unsupported.

### 6.3 Initial catch-up
- Downstream chooses `since`:
  - `0` (or early) to fetch as much as upstream allows, or
  - `received_at` of oldest packet already stored from that peer.
- Upstream:
  - **MUST** return up to `limit` historical Packets ordered by `received_at` ascending when `order = "asc"`,
  - **MUST** include a cursor and its own `received_at` in each `EVENT`,
  - **MAY** cap `limit`.
- Downstream:
  - Persists last cursor per peer subscription,
  - Re‑issues `SUBSCRIBE` with `cursor` (preferred) or `since`.
  - **MUST** persist upstream-provided `received_at` for time-based logic (age/backfill/quorum); it MAY store a local ingest timestamp separately.
- When a relay already stores a Packet and later receives the same `packet_id` via replication with an earlier `received_at`, it SHOULD update its stored `received_at` to the earliest known value. This ensures the network converges on a consistent witness time and prevents time-inflation attacks from misbehaving upstreams.

### 6.4 Resumption after disconnect
- Prefer last stored `cursor`.
- If cursor invalid/expired, fall back to `since = last upstream received_at from that peer` and rely on dedup by `packet_id`.
- If upstream returns `ERROR code: "invalid_cursor"`, downstream **MUST** fall back to `since`.

## 7. Deduplication, loops, and replay

- Relays **MUST** deduplicate by `packet_id` (RFC‑0002 §8) for at least the max backfill window.
- Replication loops are expected; they are safe if relays never modify Packets and always dedup.
- Relays **SHOULD** keep observability of which peer delivered a Packet, but MUST NOT modify Packet bodies to store that metadata.

## 8. Rate limiting, policy, and access control

Relays MAY apply policies to replication sessions:
- Per‑peer limits on concurrent replication subscriptions, historical backfill rate, and live throughput.
- Allow‑lists of relay DIDs; restrict certain profiles (e.g., `FULL_ARCHIVE`) to trusted peers.

When rate limiting:
- Upstream **MUST** reply with `ERROR code: "rate_limited"` and `retry_after` (RFC‑0004 §6.1).
- Downstream **SHOULD** back off per `retry_after`.

Local content policies (blocking authors or content types) **MUST** apply equally to user and replication subscriptions.

## 9. Replication policy documents (CONFIG)

Relays MAY publish replication policy as Packets with `content.type = "CONFIG"` and `config_type = "REPLICATION_POLICY_V1"`.

```jsonc
{
  "content": {
    "type": "CONFIG",
    "config_type": "REPLICATION_POLICY_V1",
    "relay_id": "did:strata:relay_eu1",
    "profiles": [
      {
        "name": "FULL_ARCHIVE",
        "max_backfill_seconds": 604800,
        "requires_auth": true,
        "requires_capability": "replicate"
      },
      {
        "name": "PUBLIC_SOCIAL",
        "max_backfill_seconds": 259200,
        "requires_auth": false,
        "requires_capability": null
      }
    ],
    "preferred_peers": [
      "did:strata:relay_us1",
      "did:strata:relay_apac1"
    ]
  }
}
```

Rules:
- `author_id` **MUST** equal `relay_id` when publishing its own policy.
- Clients MAY display these to users; relays MAY auto‑configure replication from them.

## 10. HTTP relay manifest

Relays **SHOULD** expose an HTTP manifest at `GET /.well-known/strata-relay.json`:

```jsonc
{
  "relay_id": "did:strata:relay_eu1",
  "ws_endpoint": "wss://eu1.relay.example/strata",
  "status": "operational",
  "policy": {
    "accepts_public_packets": true,
    "default_retention_days": 30,
    "max_backfill_seconds": 604800
  },
  "replication": {
    "accepts_peer_subscriptions": true,
    "supported_profiles": ["FULL_ARCHIVE", "PUBLIC_SOCIAL"],
    "requires_auth": true
  },
  "software": {
    "name": "strata-relay",
    "version": "0.1.0"
  }
}
```

Manifest is non-authoritative (not signed) and intended for discovery/config convenience only. Authoritative seeds remain signed Bootstrap Documents (RFC-0005/0006).

**Security Note:** Clients SHOULD NOT establish initial protocol trust, accept new relay identities for security-sensitive operations, or update pinned trust roots based solely on an unsigned HTTP manifest. Manifests MAY be used for convenience discovery (e.g., finding WebSocket endpoints), but all trust-relevant decisions MUST be validated against signed Bootstrap Documents and the Web-of-Trust.

## 11. Interaction with transport bindings

Future bindings (RFC‑0100: Strata over Nostr, RFC‑0101: Strata over ActivityPub) may define replication semantics for bridges. This RFC covers native Strata relay↔relay replication over RFC‑0004. Bridges MUST map external events into canonical Packets before applying these rules.

## 12. Security & privacy considerations

- Replication reveals relay peerings; operators may choose Tor/VPN, limit peers, or omit public manifests.
- Misbehaving peers: relays **SHOULD** blacklist peers that forward malformed Packets and maintain QoS scores (RFC‑0004 §7).
- Historical backfill: deep history can be abused for DoS; relays **MUST** enforce reasonable limits and **SHOULD** require auth for deep backfill.
