# RFC-0101: Strata over ActivityPub (Transport Binding)

- **Status:** Planned  
- **Scope:** Transport binding (normative once accepted)  
- **Author(s):** Strata Core Team  
- **Created:** 2025-12-??  

## Intent

Define how Strata Packets, provenance headers, and attestations map onto ActivityPub objects and extensions:

- Encoding the canonical Packet body inside ActivityPub activities/objects,
- Representing provenance headers and attestations as ActivityPub extensions,
- Handling signatures, addressing, and discovery in an ActivityPub federation.

This RFC will support:

- Phase 1: shipping Strata provenance and attestation data as ActivityPub extensions,
- Phase 2: dual-protocol clients/servers that speak both ActivityPub and native Strata,
- Phase 3: gradual migration toward native Strata relays where they win on UX/features.

## Notes

- Placeholder to mark the roadmap; detailed bindings will be drafted separately.
- See the whitepaper “Transport bindings & rollout” section for the staged adoption plan.
