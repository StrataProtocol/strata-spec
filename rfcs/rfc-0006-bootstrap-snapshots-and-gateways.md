<!-- rfcs/rfc-0006-bootstrap-snapshots-and-gateways.md -->

# RFC-0006: Bootstrap Snapshots and Dynamic Gateways

- **Status:** Draft  
- **Author(s):** Strata Core Team (proposed)  
- **Created:** 2025-11-27  
- **Updated:** 2025-12-25  
- **Scope:**  
  - **Normative:** `CONFIG` / `BOOTSTRAP_SNAPSHOT` packet shape and client verification rules  
  - **Non‑normative:** Dynamic bootstrap gateway roles and deployment patterns  

---

## 1. Summary

This RFC defines:

1. A **standard `CONFIG` packet subtype** called a **Bootstrap Snapshot** (`scope = "BOOTSTRAP_SNAPSHOT"`), which carries a fresh, aggregated view of:
   - relays,
   - seed gatekeepers,
   - hardware and model roots,
   - optional DID documents and trust edges.

2. An **optional deployment pattern** called a **Dynamic Bootstrap Gateway**:
   - a service that periodically publishes Bootstrap Snapshots and may expose them over HTTP or relays.

Bootstrap Snapshots are **advisory discovery inputs**. They do **not** introduce new roots of trust and do **not** replace **Bootstrap Documents** signed by bootstrap gatekeepers as defined in the trust/reputation reference RFC.

Clients **MAY** consume Bootstrap Snapshots from any source (including gateways), but **MUST** still:

- validate Bootstrap Documents using pinned bootstrap gatekeeper keys, and  
- be able to bootstrap from at least one static Bootstrap Document without any gateway.  

---

## 2. Motivation

Brand‑new Strata clients need:

- A way to discover **relays**,  
- A baseline set of **seed gatekeepers** for seed trust,  
- **Hardware/model roots** for provenance chains.

Bootstrap Documents already provide this, but they are:

- **Static** (ship with the client or are obtained out‑of‑band),  
- **Slow to evolve** (require app updates or explicit user action).

Many deployments want something like **DNS seeds**:

- Zero‑day, somewhat centralized snapshot of “healthy” relays and identities,
- Updatable without shipping a new app,
- Still anchored in the same Strata trust model.

This RFC standardizes **how such snapshots are represented and validated** (normative) while keeping the notion of “Dynamic Bootstrap Gateways” **optional and non‑normative**, to avoid introducing mandatory central choke points.

---

## 3. Relationship to Other RFCs

This RFC builds on:

- **RFC‑0000 – Terminology and Conventions**  
  Normative language, identifier formats, `content.type = "CONFIG"`.
- **RFC‑0001 – Identity and StrataID**  
  DID format, DID resolution, signature verification.
- **RFC‑0002 – Data Packets and Provenance Headers**  
  Packet structure, canonicalization, `packet_id`, `content`, `CONFIG`.
- **RFC‑0004 – Relay Transport Protocol**  
  Relay publish/subscribe, `content.type` handling, `received_at`.
- **RFC‑0005 – Trust, Reputation, and Reality Tuner** (reference)  
  Defines Bootstrap Documents, bootstrap gatekeepers, seed trust.

Where this RFC conflicts with non‑normative text in RFC‑0005, the normative requirements in this RFC take precedence.

---

## 4. Terminology

**Bootstrap Document**  
Content‑addressed, signed configuration object that bootstraps:

- relay list,
- seed gatekeepers,
- hardware roots,
- model roots.

**Bootstrap Gatekeeper Key**  
Public key(s) baked into a client, used solely to verify Bootstrap Documents and their diversity requirements.

**Bootstrap Snapshot**  
A Packet with `content.type = "CONFIG"` and `content.scope = "BOOTSTRAP_SNAPSHOT"` that aggregates current configuration information relevant to initial network discovery (this RFC, §5).

**Dynamic Bootstrap Gateway**  
A service that periodically computes and publishes Bootstrap Snapshots and may expose them via HTTP, relays, or other transports. Gateways **do not** define new trust roots; they are discovery accelerators only.

---

## 5. Bootstrap Snapshot Packet (Normative)

### 5.1 Packet Shape

A **Bootstrap Snapshot** is a normal Data Packet as in RFC‑0002 with:

```jsonc
{
  "packet_id": "0x...",          // per RFC-0002
  "version": 1,
  "timestamp": 1715421200,
  "author_id": "did:strata:bootstrap_gateway_x",
  "content": {
    "type": "CONFIG",
    "scope": "BOOTSTRAP_SNAPSHOT",
    "snapshot_version": 1,
    "issued_at": 1715421200,
    "expires_at": 1715424800,

    "source_id": "did:strata:bootstrap_gateway_x",
    "bootstrap_docs": [
      {
        "bootstrap_id": "0x...",
        "priority": 100,
        "reason": "current_default"
      }
    ],

    "relays": [
      {
        "relay_id": "did:strata:relay_eu1",
        "url": "wss://relay.eu1.example/strata",
        "score": 0.92,
        "tags": ["region:eu", "profile:default"]
      }
    ],

    "seed_gatekeepers": [
      {
        "id": "did:strata:seed_foundation",
        "label": "Strata Foundation",
        "weight": 1.0,
        "tags": ["org:strata_foundation"]
      }
    ],

    "hardware_roots": [
      {
        "id": "did:hardware-root:2025",
        "tags": ["vendor:vendor_x"]
      }
    ],

    "model_roots": [
      {
        "id": "did:model-root:2025",
        "tags": ["provider:model_lab_y"]
      }
    ],

    "trusted_capture_apps": [
      {
        "id": "did:strata:official_camera_app",
        "platform": "ios",
        "bundle_id": "com.strata.camera",
        "min_version": "1.0.0",
        "attestation_required": true,
        "tags": ["official", "verified"]
      },
      {
        "id": "did:strata:official_camera_app_android",
        "platform": "android",
        "bundle_id": "com.strata.camera",
        "min_version": "1.0.0",
        "attestation_required": true,
        "tags": ["official", "verified"]
      }
    ],

    "did_docs": [
      {
        "did": "did:strata:seed_foundation",
        "did_doc_hash": "z6M...",
        "priority": 50
      }
    ],

    "trust_edges": [
      {
        "from": "did:strata:seed_foundation",
        "to": "did:strata:journalists_guild",
        "edge_packet": "0xedge1",
        "weight_hint": 0.9
      }
    ]
  },
  "signature": "0x..."
}
```

Fields:

- `content.type` **MUST** be `"CONFIG"`.  
- `content.scope` **MUST** be `"BOOTSTRAP_SNAPSHOT"`.
- `snapshot_version` – integer, **MUST** be present; versioning for this structure.
- `issued_at` – Unix epoch seconds; **MUST** be present.
- `expires_at` – Unix epoch seconds; **MUST** be present; see §5.3.

- `source_id` – StrataID of the entity producing the snapshot:
  - **MUST** equal the Packet’s `author_id`.
  - **MUST** be a valid DID per RFC‑0001.

- `bootstrap_docs` – optional array:
  - `bootstrap_id` – content‑addressed ID of a Bootstrap Document.
  - `priority` – numeric hint (higher = preferred).
  - `reason` – human‑readable reason; optional.

- `relays` – optional array of relay recommendations:
  - `relay_id` – relay StrataID (DID).  
  - `url` – WebSocket or other URI.  
  - `score` – advisory QoS/reputation hint in [0,1]; optional.
  - `tags` – optional tags (region, profile, etc.).

- `seed_gatekeepers` – optional array:
  - `id` – StrataID to be used as seed gatekeeper.  
  - `label` – optional human-friendly name (advisory UI hint only).  
  - `weight` – advisory weight.
  - `tags` – e.g., `["org:ngo", "region:us"]`.

- `hardware_roots` / `model_roots` – optional arrays:
  - `id` – DID of a hardware/model root.
  - `tags` – optional descriptive tags.

- `trusted_capture_apps` – optional array of trusted capture application identities:
  - `id` – StrataID (DID) of the capture application.
  - `platform` – Target platform: `ios`, `android`, `web`, `desktop`, or `*` for all.
  - `bundle_id` – Platform‑specific app identifier (iOS bundle ID or Android package name).
  - `min_version` – Minimum trusted version (semver string).
  - `attestation_required` – Whether platform attestation (`apple-app-attest` or `play-integrity`) is required for elevated trust.
  - `tags` – optional descriptive tags (e.g., `["official", "verified"]`).

- `did_docs` – optional array of DID Document pointers:
  - `did` – target DID.
  - `did_doc_hash` – content‑addressed hash per RFC‑0001.
  - `priority` – advisory preference.

- `trust_edges` – optional array of trust edge references:
  - `from`, `to` – DIDs.
  - `edge_packet` – `packet_id` of a trust-edge packet.
  - `weight_hint` – optional hint.

Implementations **MUST** ignore unknown fields in the `CONFIG` object, per RFC‑0000 JSON rules.

### 5.1.1 Advisory Nature of Capture App Lists

The `trusted_capture_apps` field is **strictly advisory**. To preserve user autonomy and prevent centralized gatekeeping (per Axioms 4.2, 4.5, 4.6):

1. **User Control**: Clients **MUST** allow users to:
   - View, edit, disable, or extend the trusted capture apps list.
   - Override snapshot-provided lists with their own preferences.
   - Disable capture app trust entirely.

2. **Trust Levels**: Clients **MUST NOT**:
   - Treat missing app attestation as "untrusted" (red ring).
   - Automatically hide or filter content lacking app attestation.
   - Require app attestation for content to be visible.

3. **Signal Weight**: Verified capture app attestation **SHOULD**:
   - Be treated as ONE signal among many (not a guarantee).
   - Contribute to yellow-green trust level, not automatic green.
   - Be clearly labeled in UI as "Verified capture app" rather than implying absolute trust.

4. **Transparency**: Clients **SHOULD**:
   - Display which capture apps are trusted and why.
   - Show provenance details including app attestation status.
   - Allow users to inspect the full attestation chain.

This ensures capture app trust enhances rather than replaces the pluralistic attestation model. Hardware/app attestations provide evidence, not verdicts.

### 5.2 Signature and Validation

Bootstrap Snapshot Packets:

1. **MUST** be canonicalized and hashed as per RFC‑0002 to derive `packet_id`.  
2. **MUST** be signed by a key authorized for `author_id` (via DID Document and key events).  
3. **MUST** be validated as ordinary Packets before any content is used.

Clients **MUST NOT**:

- Treat Snapshot contents as trusted unless the Packet passes all standard signature and key‑validity checks.
- Accept `source_id` that does not equal `author_id`.

This RFC **does not** grant any special trust semantics to Snapshot authors; trust in `source_id` is determined via the normal trust/reputation mechanisms (e.g., RFC‑0005).

### 5.3 Freshness and Expiry

Clients **MUST** enforce:

- `issued_at` **MUST NOT** be more than the global `clock_skew_seconds` in the future relative to local clock.  
- `expires_at` **MUST** be greater than `issued_at`.
- A Snapshot **MUST NOT** be used if:
  - `now > expires_at + clock_skew_seconds`, or  
  - it fails Packet freshness rules in RFC‑0002 (e.g., stale `timestamp`).  

Snapshot producers **SHOULD**:

- Use relatively short lifetimes (e.g., tens of minutes to a few hours) to reflect the dynamic nature of the view.
- Publish updated Snapshots well before `expires_at`.

### 5.4 Relationship to Bootstrap Documents

Clients **MUST** treat `bootstrap_docs` entries in a Snapshot as **pointers or recommendations only**.

For each candidate `bootstrap_id`:

1. Clients **MUST** fetch the corresponding Bootstrap Document (over relays, HTTP, or local cache).  
2. Clients **MUST** validate it against the pinned bootstrap gatekeeper keys and diversity rules from RFC‑0005 (signature threshold, diversity tags). For production clients, the minimum quorum is **at least 3 distinct gatekeeper signatures** and **at least 2 distinct diversity tags** (RFC‑0005 §4.1); clients **MUST** fail closed if quorum is not met. Development/test builds **MAY** relax thresholds but **MUST** surface a clear warning and **MUST NOT** treat relaxed Bootstrap Documents as production‑trust.  
3. Only after successful validation **MAY** clients treat the referenced Bootstrap Document as authoritative for:
   - relay lists,
   - seed gatekeepers,
   - hardware roots,
   - model roots.

A Snapshot **MUST NOT** be able to “bless” a Bootstrap Document that fails bootstrap‑key verification.

---

## 6. Client Behavior (Normative)

### 6.1 Offline Baseline Requirement

Every conformant client implementation **MUST** support the ability to bootstrap using **at least one static Bootstrap Document** obtained without any dynamic gateway (e.g., shipped with the client, configured by the user, or provided out‑of‑band).

If all dynamic sources (gateways, remote relays) are unavailable, the client **MUST** still be able to:

- Verify a static Bootstrap Document using pinned bootstrap gatekeeper keys, and  
- Discover at least one relay from that document.

### 6.2 Optional Use of Bootstrap Snapshots

Clients **MAY** implement support for Bootstrap Snapshots. If they do, they **MUST**:

1. Validate each Snapshot Packet as in §5.2.
2. Reject Snapshots that are not fresh per §5.3.
3. Treat all Snapshot content as **advisory**:
   - `relays` suggestions MAY be used to prioritize or discover relays but MUST NOT override explicit user configuration.
   - `seed_gatekeepers`, `hardware_roots`, and `model_roots` MUST be cross‑checked against validated Bootstrap Documents before affecting trust engine behavior.
   - `trusted_capture_apps` **MUST** be user-editable and **MUST NOT** be required for content visibility. See §5.1.1.

### 6.3 Multi‑Snapshot and Quorum Handling (Recommended)

When multiple Snapshots are available (from different `source_id`s):

- Clients **SHOULD** fetch and compare Snapshots from **at least two** distinct sources where possible.
- Clients **SHOULD** prefer configuration elements (e.g., relays, bootstrap_docs) that appear in Snapshots from multiple, independently trusted sources.
- Conflicts (e.g., one Snapshot recommending a relay as “good” and another omitting or down‑weighting it) **SHOULD** be surfaced to advanced users and **MAY** be resolved using local reputation of `source_id`s.

This section is **RECOMMENDED** but not strictly required for interoperability.

### 6.4 Failure Modes

If no valid Snapshot is available (e.g., all are expired, invalid, or unreachable), clients **MUST**:

- Fall back to static Bootstrap Documents and any previously cached configuration,
- Not treat the absence of a Snapshot as a hard error.

---

## 7. Dynamic Bootstrap Gateways (Non‑Normative)

This section is **non‑normative** and describes a recommended deployment pattern.

### 7.1 Role

A **Dynamic Bootstrap Gateway**:

- Runs as a service (often by foundations, NGOs, or ecosystem operators),  
- Periodically queries relays, Bootstrap Documents, DID registries, etc.,  
- Computes a “network snapshot” including:
  - known healthy relays,
  - recommended Bootstrap Documents,
  - frequently used DID docs and trust edges,
- Publishes that view as one or more **Bootstrap Snapshot Packets**.

Gateways behave as **advisory aggregators**, not protocol authorities.

### 7.2 Recommended HTTP Interface

Gateways are RECOMMENDED to expose a simple HTTP API in addition to publishing Snapshots as Packets, for use by first‑run clients that are not yet connected to any relays:

- `GET /.well-known/strata/bootstrap-snapshot`  
  Returns the latest Snapshot in JSON (same shape as the Packet from §5.1) plus metadata:
  ```jsonc
  {
    "packet": { /* full Packet including signature */ },
    "observed_at": 1715421205
  }
  ```

- `GET /.well-known/strata/bootstrap-snapshots?limit=N`  
  Returns recent Snapshot history, enabling clients to see changes over time.

Clients using this HTTP API **MUST** still verify the returned Packet as if it had been received over a relay.

### 7.3 Discovery of Gateways

Bootstrap Documents **MAY** be extended to include a `bootstrap_gateways` field:

```jsonc
"bootstrap_gateways": [
  {
    "id": "did:strata:bootstrap_gateway_x",
    "https": "https://bootstrap.x.example",
    "relays": [
      "wss://relay.x.example/strata"
    ],
    "tags": ["profile:default", "region:eu"]
  }
]
```

This field is **advisory**:

- Clients **MAY** query these endpoints on first run,
- Clients **MAY** ignore them entirely and rely on their own configuration.

The presence of a gateway in a Bootstrap Document does **not** elevate it to a trust root; it is simply a suggested discovery service.

### 7.4 Example First‑Run Flow (Non‑Normative)

1. Client loads with:
   - pinned bootstrap gatekeeper keys,
   - one or more static Bootstrap Documents (or their IDs).

2. Client validates at least one Bootstrap Document.

3. From the Bootstrap Document, client learns:
   - base relays,
   - optional `bootstrap_gateways`.

4. In parallel:
   - Connect to base relays.  
   - Query multiple gateways’ `/.well-known/strata/bootstrap-snapshot`.

5. For each Snapshot:
   - Verify Packet signature and freshness.
   - Merge relay lists, Bootstrap Document recommendations, DID docs.

6. Build an initial configuration:
   - Start with static bootstrap config,
   - Overlay “consensus” elements from Snapshots that agree across multiple sources.

7. Proceed with normal Strata behavior (trust engine, feeds, etc.).

---

## 8. Security & Privacy Considerations

### 8.1 Centralization Risk

- Over‑reliance on a small set of gateways can create de‑facto central choke points, contrary to Strata’s goals as a decentralized verification layer.  
- Mitigations:
  - Treat gateways as **optional** (this RFC enforces that via §6.1),
  - Query multiple independent gateways where feasible,
  - Allow users and communities to run their own gateways.

### 8.2 Snapshot Poisoning

An attacker controlling a gateway could:

- Recommend malicious relays or omit honest ones,
- Recommend obsolete or malicious Bootstrap Documents (that still verify).

Mitigations:

- Clients **MUST** still validate all Bootstrap Documents via pinned bootstrap gatekeeper keys (§5.4).  
- Clients **SHOULD** use local reputation and Web‑of‑Trust to decide how much weight to give each `source_id`.  
- Cross‑checking Snapshots across multiple gateways is RECOMMENDED (§6.3).

### 8.3 Privacy

- Querying a gateway directly leaks some metadata (IP, timing, user‑agent).
- For sensitive users, clients MAY:
  - Fetch Snapshots over privacy‑enhancing transports (Tor, VPN),
  - Prefer Snapshots received over relays instead of direct HTTP.

---

## 9. Backwards Compatibility

- Older clients that do not implement this RFC will simply ignore `CONFIG` Packets with unknown `scope` values, per RFC‑0000/0002 rules.  
- Gateways can safely publish Bootstrap Snapshots today without breaking existing relays or clients.
- Future RFCs may define additional `CONFIG` scopes; implementers **MUST** ignore unknown scopes while preserving packets.
