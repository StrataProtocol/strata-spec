# Strata Protocol – Specification

This repository contains the **canonical specification** for the Strata Protocol:

- The high-level **whitepaper** (vision + architecture + draft spec),
- A series of focused **RFCs** (protocol details, data structures, transport, reputation, etc.).

Implementers of relays, clients, SDKs, and attestor services should treat this repo as the source of truth.

---

## Repository Structure

```text
strata-spec/
  README.md
  whitepaper.md
  rfcs/
    rfc-0000-terminology-and-conventions.md
    rfc-0001-identity-and-strataid.md
    rfc-0002-packets-and-provenance.md
    rfc-0003-attestations-and-retroactive-consensus.md
    rfc-0004-relay-transport-protocol.md
    rfc-0005-trust-reputation-and-reality-tuner.md
```
### Top-Level
#### whitepaper.md
Complete narrative + technical overview:
- Problem, vision, and “Strata Axioms”
- Architecture (Identity, Provenance, Relays, Reality Tuner)
- Threat model and design goals
- Draft protocol specification (Packet schema, relay API, reputation model)
- Implementation roadmap

#### RFCs

RFCs are focused, append-only design documents.
They go deeper than the whitepaper in specific areas.

##### `rfcs/rfc-0000-terminology-and-conventions.md`
Shared vocabulary and rules:
- Core terms: StrataID, Packet, Relay, Attestation, Genesis Event, Reality Tuner
- JSON + canonicalization rules
- Hashing, time formats (Unix seconds), enum semantics
- RFC 2119 normative language conventions

##### `rfcs/rfc-0001-identity-and-strataid.md`

Identity model:
- `did:strata` method (StrataIDs)
- Key generation, storage, rotation, recovery (conceptual)
- Multiple personas per human
- Trust edges between identities (Web-of-Trust base layer)

##### `rfcs/rfc-0002-packets-and-provenance.md`

Core data model:
- Data Packet structure (`packet_id`, `content`, `provenance_header`, `attestations`, etc.)
- Canonicalization and hashing rules
- Provenance header (`origin_type`, `device_id`, `edit_history`)
- Genesis Events for media

##### `rfcs/rfc-0003-attestations-and-retroactive-consensus.md`

Attestation system:
- Attestation objects (embedded + standalone)
- Claim types (provenance analysis, fact-checking, spam/abuse)
- Attestor types (`CLIENT`, `NGO`, `LAB`, `MEDIA`, `MODEL_PROVIDER`, `OTHER`)
- Retroactive consensus (quorum over attestations, anti-viral downgrade behavior)

##### `rfcs/rfc-0004-relay-transport-protocol.md`

Relay layer:
- Minimal WebSocket protocol (SUBSCRIBE, PUBLISH, EVENT, OK, ERROR)
- Relay responsibilities and non-responsibilities
- Storage/indexing expectations
- Local policy hooks (rate limiting, blocking, legal compliance)

##### `rfcs/rfc-0005-trust-reputation-and-reality-tuner.md`
Trust + filtering reference model:
- Composite reputation (seed trust, Web-of-Trust, stake, behavior)
- Sybil resistance (age-gating, trust budgets, cluster down-weighting)
- Quorum logic for claims over attestations
- Reality Tuner modes (Strict / Standard / Wild)
- Mapping signals → traffic-light rings (🟢/🟡/🔴)

## Who Should Read What?

### Protocol / SDK engineers
- Start with: whitepaper.md
- Then: rfc-0000, rfc-0001, rfc-0002
- For relays: add rfc-0004
- For Reality Tuner / ranking: add rfc-0005
- For attestor services: add rfc-0003

### Product / UX / strategy
- Read: whitepaper.md
- Skim: rfc-0005 (Reality Tuner), rfc-0003 (attestations)

### Researchers / auditors
- Read: whitepaper.md
- Then all RFCs, especially 0000, 0002, 0003, 0005.

### RFC Process
Strata uses a lightweight RFC process:
- Open an issue
- Describe the problem / design space.
- Draft an RFC under rfcs/
- Name: rfc-XXXX-short-title.md (next available number).
- Start with Status: Draft.
- Submit a PR
- Tag relevant maintainers.
- Expect async comments / iteration.

### Status changes
- Draft → Active once accepted.
- Superseded if replaced by a later RFC.
- Informational for non-normative notes.

**RFCs SHOULD be backward compatible when possible; breaking changes MUST be clearly documented.**

### Versioning & Compatibility
#### Protocol stability:

The spec is currently experimental. Breaking changes are likely while Strata is pre-1.0.

#### Implementations:
Implementers SHOULD:
- Pin to a specific commit / tag of strata-spec,
- Note which RFC versions they support,
- Expose this in their own README / docs.
- Once the protocol matures, we’ll move toward semver-style versioning for the spec.

### Contributing
Contributions are welcome.

- For typos / clarifications: Open a small PR directly.
- For new features / behavior changes: Open an issue first,
Propose an RFC or update to an existing one.

### General guidelines:
- Keep RFCs focused and self-contained.
- Use examples liberally (JSON snippets, flows).
- Use RFC 2119 language (MUST, SHOULD, MAY) where behavior is normative.

### License
TODO: Choose and document license for the spec
(e.g. Creative Commons Attribution 4.0 / Apache-2.0 for code examples).
Until explicitly specified otherwise, treat this repo as not yet licensed for reuse outside the Strata project. This will be updated before public release.

### Status
This spec is early-stage and will evolve rapidly as:
- The reference relay (strata-relay) and client (strata-client) are implemented,
- SDKs (e.g. strata-sdk-ts) solidify the data model,
- Real-world adversarial testing informs reputation + provenance design.

Expect breaking changes; do not assume long-term stability yet.