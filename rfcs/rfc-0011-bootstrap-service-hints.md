# RFC-0011: Bootstrap Service Hints

- **Status:** Draft
- **Author(s):** Strata Core Team
- **Created:** 2025-12-15
- **Updated:** 2025-12-15
- **Scope:** Normative protocol (Bootstrap Snapshot extension + service config packets)

## 1. Summary

This RFC defines a generic, extensible way to advertise **service configuration hints**
through Bootstrap Snapshots (RFC-0006). It introduces:

- A `hints.services` array inside Bootstrap Snapshot `CONFIG` payloads.
- A generic `SERVICE_CONFIG` packet scope for provider-signed service configs.

The goal is to enable production-ready discovery of auxiliary services (e.g., ICE for
WebRTC) without creating new trust roots or central gatekeepers.

## 2. Motivation

Some services required for real-time or high-reliability experiences (e.g., WebRTC
TURN/STUN, push relays, media pinning) are not discoverable via the core relay mesh.
Bootstrap Snapshots already provide advisory relay lists; adding generic service hints
provides a similar discovery path while preserving user control and decentralization.

## 3. Terminology

- **Snapshot**: A Bootstrap Snapshot Packet (RFC-0006).
- **Service hint**: An advisory hint describing a service provider and config.
- **Provider**: The operator of a service, identified by a Strata DID.

## 4. Bootstrap Snapshot Extension (Normative)

Bootstrap Snapshots MAY include an optional `hints.services` array under `content`.

```jsonc
{
  "type": "CONFIG",
  "scope": "BOOTSTRAP_SNAPSHOT",
  "hints": {
    "services": [
      {
        "kind": "ICE",
        "version": 1,
        "provider_id": "did:strata:icepool_a",
        "expires_at": 1715429999,
        "priority": 50,
        "config_packet": "0x...",
        "payload": { ... }
      }
    ]
  }
}
```

Fields:

- `kind` (required): uppercase service identifier.
- `version` (required): integer service hint version.
- `provider_id` (required): provider DID.
- `expires_at` (required): unix seconds when the hint is no longer valid.
- `priority` (optional): advisory priority (higher preferred).
- `config_packet` (optional): `packet_id` of a provider-signed SERVICE_CONFIG Packet.
- `payload` (optional): inline service config when `config_packet` is absent.

Rules:

- Unknown fields MUST be ignored (RFC-0000).
- Hints MUST be treated as advisory and MUST NOT override explicit user settings.
- Clients MUST ignore hints where `expires_at` is in the past.
- Clients SHOULD ignore hints where `expires_at` is more than 7 days after the
  Snapshot `issued_at`.
- If `config_packet` is present, `payload` MAY be omitted.
- If `config_packet` is absent, `payload` MUST be present.

## 5. SERVICE_CONFIG Packet (Normative)

Providers MAY publish a signed config Packet for a given service.

```jsonc
{
  "content": {
    "type": "CONFIG",
    "scope": "SERVICE_CONFIG",
    "service": "ICE",
    "service_version": 1,
    "provider_id": "did:strata:icepool_a",
    "issued_at": 1715421000,
    "expires_at": 1715429999,
    "payload": { ... }
  }
}
```

Rules:

- `content.type` MUST be `"CONFIG"`.
- `content.scope` MUST be `"SERVICE_CONFIG"`.
- `provider_id` MUST match the Packet `author_id`.
- `service` and `service_version` MUST match the referencing hint.
- `payload` schema is service-specific (see Section 7).

## 6. Validation and Auto-Enable Rules (Normative)

Service hints are advisory and MAY be ignored.

For automated use without explicit user opt-in, clients MUST:

1. Require a valid provider-signed `SERVICE_CONFIG` Packet referenced by `config_packet`.
2. Require the hint (or the referenced config packet) to be observed in at least
   two distinct Snapshots with different `source_id` values within the last 72 hours.

Inline `payload` without a `config_packet` MUST NOT be auto-enabled. It MAY be exposed
in advanced settings for explicit user opt-in.

Clients SHOULD de-duplicate providers by URL and rotate across providers to avoid
central lock-in.

## 7. Service Payloads (Registry)

`payload` is a JSON object whose schema is defined per service `kind`. This RFC
defines the registry concept but does not mandate a schema for each service.

Initial registry entries (informational):

- `ICE` - WebRTC ICE server hints (defined by application-specific specs such as
  StrataX IS-0008 or future RFCs).
- `PUSH` - Push notification relays (future).
- `STORAGE` - Media pinning or storage services (future).

## 8. Security and Privacy Considerations

- Service hints are not trust roots; clients MUST retain user control.
- Automated use MUST require provider signatures and multi-source observation.
- Clients SHOULD minimize logging of service endpoints and avoid prefetching
  credentials unless required.
- Providers SHOULD issue short-lived configs and rotate endpoints as needed.

## 9. Backwards Compatibility

This RFC is fully backwards compatible. Clients and relays that do not implement
these hints will ignore the new fields per RFC-0000 and continue to operate normally.
