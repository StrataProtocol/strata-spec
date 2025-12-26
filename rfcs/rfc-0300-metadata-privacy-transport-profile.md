<!-- rfcs/rfc-0300-metadata-privacy-transport-profile.md -->

# RFC-0300: Metadata Privacy Transport Profile (Optional)

- **Status:** Draft  
- **Scope:** Optional transport profile (non-normative until accepted)  
- **Author(s):** Strata Core Team  
- **Created:** 2025-12-25  

## 1. Summary

This RFC defines an optional transport profile that reduces metadata exposure while remaining compatible with RFC-0004. It does not change Packet formats, relay messages, or signature rules. Instead, it specifies client behaviors and relay hygiene that reduce how much any single observer learns.

This profile is opt-in and must not be presented as full anonymity. It is a best-effort mitigation layer for high-risk users.

## 2. Motivation

RFC-0004 states that relays and network observers can see metadata even when content is encrypted. That makes traffic analysis possible and limits user safety in high-risk contexts. This RFC documents practical mitigations that can be implemented without breaking interoperability.

## 3. Goals

- Provide a compatibility profile that clients can implement without new wire formats.
- Reduce metadata exposure to any single relay or observer.
- Make tradeoffs explicit in UI and documentation.
- Preserve compatibility with existing relays and clients.

## 4. Non-goals

- No guarantee of anonymity or unlinkability against a global observer.
- No changes to RFC-0004 message types or Packet schemas.
- No mandatory relay infrastructure changes.

## 5. Threat Model

Assumes the following adversaries:

- Relay operators observing connection metadata and subscription shapes.
- Network observers correlating IP addresses, timing, and packet sizes.
- Colluding relays sharing logs.

Out of scope:

- Compromised client devices.
- Global adversaries with full visibility of all relay links.

## 6. Profile Levels

This RFC defines two optional profiles. Clients MAY implement one or both. Relays do not need to change behavior to support them.

### 6.1 Profile: SHIELDED (client-only)

The SHIELDED profile reduces metadata exposure through client behavior alone.

Clients implementing SHIELDED:

- MUST use at least two relays for read and publish paths.
- MUST avoid single-relay dependency for any feed or publish path.
- SHOULD use coarser subscription filters where possible and apply finer filters client-side.
- SHOULD randomize publish timing within a bounded jitter window.
- SHOULD rotate or separate identities by context (pseudonymous DIDs) for high-risk usage.
- SHOULD limit or disable relay auth if it is not required for the intended path.

### 6.2 Profile: HARDENED (client plus network path)

The HARDENED profile adds network path privacy on top of SHIELDED behaviors.

Clients implementing HARDENED:

- MUST satisfy all SHIELDED requirements.
- MUST use a privacy-preserving network path for relay connections (e.g., VPN, Tor, or a trusted proxy).
- SHOULD avoid stable IP reuse across sessions and identities.
- SHOULD batch subscriptions and publishes to reduce timing correlation.

### 6.3 Client mapping and defaults (informative)

Clients MAY use user-friendly labels as long as they map to these profile requirements.

- **STANDARD:** Default behavior per RFC-0004 with no extra privacy guarantees.
- **EXTRA PRIVACY:** Maps to SHIELDED (multi-relay + jitter + minimized auth).
- **MAXIMUM PRIVACY:** Maps to HARDENED (SHIELDED + privacy-preserving network path).

Implementations SHOULD default to STANDARD and avoid implying anonymity.

### 6.4 Implementation checklist (informative)

Clients implementing SHIELDED or HARDENED SHOULD:

- Use a minimum of 2 relays (SHIELDED) or 3 relays (HARDENED) for read + publish.
- Apply publish timing jitter to reduce correlation. Recommended ranges:
  - SHIELDED: 200–1200 ms
  - HARDENED: 800–2500 ms
- Limit relay auth to the primary relay when feasible.
- Keep subscription filters coarse (types, broad time windows) and apply finer filtering client-side.
- Avoid session-stable identifiers where not required (e.g., reuse of auth tokens across relays).

Clients SHOULD surface clear UI labels for the active profile and provide a quick exit to STANDARD.

## 7. Relay Guidance

Relays that wish to support privacy-focused clients:

- SHOULD minimize retention of connection logs and IP addresses.
- SHOULD document their log retention policy and operational controls.
- SHOULD accept connections from privacy-preserving network paths.
- SHOULD avoid requiring long-lived auth tokens when not needed.

This RFC does not mandate relay-side changes or new message types.

## 8. UX Requirements

Clients exposing these profiles:

- MUST display a clear notice that metadata privacy is limited.
- MUST explain tradeoffs (latency, bandwidth, availability).
- SHOULD mark profiles as experimental and place them behind advanced/developer settings until accepted.
- SHOULD allow per-identity or per-session toggles.

## 9. Compatibility

This profile is fully compatible with RFC-0004:

- No new WebSocket message types are introduced.
- Packets remain canonical and are validated the same way.
- Relays that ignore this RFC still interoperate.

### 9.1 Compatibility matrix

| Capability | Clients | Relays |
|-----------|---------|--------|
| Multi-relay connections (SHIELDED/HARDENED) | MUST | N/A |
| Publish timing jitter | SHOULD | N/A |
| Coarse subscriptions + client-side filtering | SHOULD | N/A |
| Auth minimization (avoid auth when not required) | SHOULD | SHOULD allow unauth where policy permits |
| Privacy network path (HARDENED) | MUST | SHOULD accept VPN/Tor/proxy traffic |
| Log retention minimization | N/A | SHOULD |

### 9.2 Downgrade behavior

Clients MUST handle missing capabilities without breaking interoperability:

- If the configured relay pool cannot meet the minimum relay count, clients MUST fall back to STANDARD behavior.
- If HARDENED is selected but no privacy network path is available, clients MUST fall back to SHIELDED and warn the user.
- Downgrades SHOULD be session-scoped and MUST NOT silently rewrite stored user preferences.
- Clients SHOULD surface a clear, plain-language notice when a downgrade is active.

## 10. Security and Privacy Considerations

- These profiles reduce metadata exposure but cannot prevent correlation by a global observer.
- Multi-relay use reduces reliance on any single relay but increases traffic exposure overall.
- Clients SHOULD avoid implying that privacy profiles provide anonymity.

## 11. Validation (informative)

Implementations SHOULD document validation efforts that demonstrate expected effects, such as:

- Publish jitter distribution (min/max bounds, variance over time).
- Relay coverage (how many relays receive publishes per session).
- Auth scope behavior (which relays received auth vs unauth traffic).

These measurements do not guarantee anonymity; they document expected behavior and tradeoffs.

## 12. Future Work

Potential extensions that are out of scope for this RFC:

- Cover traffic and batching at the relay level.
- Blinded auth tokens for relay access.
- Mixnet or onion routing profiles with stronger anonymity guarantees.
- Relay capability signaling for privacy support.
