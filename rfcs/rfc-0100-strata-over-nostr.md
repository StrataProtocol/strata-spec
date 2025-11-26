# RFC-0100: Strata over Nostr (Transport Binding)

- **Status:** Planned  
- **Scope:** Transport binding (normative once accepted)  
- **Author(s):** Strata Core Team  
- **Created:** 2025-12-??  

## Intent

Define how Strata Packets, provenance headers, and attestations are carried over Nostr:

- Event kinds / tags for embedding the canonical Packet body,
- Mapping of provenance headers into event content/tags,
- Guidance for relay/client interoperability and signature verification.

This RFC will enable:

- Phase 1 adoption: emitting Strata Packets as Nostr events without requiring native Strata relays,
- Phase 2: dual-protocol clients that speak both Strata Relay (RFC‑0004) and Nostr,
- Phase 3: coexistence with native Strata relays competing on UX/features.

## Notes

- This is a placeholder to signal roadmap intent; detailed wire mappings will be drafted in a future PR.
- See the whitepaper “Transport bindings & rollout” section for the staged deployment strategy.
