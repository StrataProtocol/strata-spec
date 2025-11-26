# Protecting Children Without Building a Mass-Surveillance Machine

How Strata Protocol can help Europe square child protection, safety, and privacy.

## 1. The moment we're in

- EU member states have agreed on a risk-based Child Sexual Abuse Regulation (CSAR) and a new EU Centre on Child Sexual Abuse to coordinate takedowns and indicators (source: Council/Consilium reports).
- The Council text still leaves room for "voluntary" scanning of private communications and makes current chat-scanning exemptions permanent — described by researchers and civil society as de facto mass surveillance that threatens end-to-end encryption (source: Tech press and MEP statements).
- Experts warn that generalized client-side scanning is technically fragile, easy for determined abusers to evade, and creates attack surfaces that invite repurposing beyond child protection (source: Global Encryption Coalition).

This sets up a false binary:
- Protect children and accept pervasive scanning, or
- Protect privacy and accept that kids are unsafe.

Strata exists to break that binary.

## 2. What Strata is (and isn't)

Strata is a decentralized verification layer — cryptographic provenance, attestations, and user-controlled filtering on top of existing apps and feeds.

Four layers:
- **L0 Identity & Trust** — StrataIDs (DIDs) and trust edges between them.
- **L1 Provenance** — Packets with a `provenance_header` and optional Genesis Events describing how media was created and edited.
- **L2 Transport Mesh** — "dumb" relays that move encrypted Packets; they do not build feeds or decide what is allowed.
- **L3 Applications & Trust Engine** — clients that implement feeds plus the local policy engine (Reality Switch presets) to decide what to show, blur, or hide.

Notably:
- Strata is not a truth oracle or centralized moderator; it does not impose protocol-level bans.
- Strata is not a new walled garden; it is a set of open RFCs any app or relay can adopt.

That fits the EU's risk-based approach: high-risk services get stronger child-safety controls without blanket scanning of private messages.

## 3. How Strata helps with child protection

### 3.1 Safer defaults for kids: device-side, parent-controlled

- Clients ship with a three-position Reality Switch (Strict, Standard, Wild). Strict can become **Kids Mode**:
  - Prefer strong provenance (hardware-captured, well-known sources).
  - Down-rank or hide content flagged as abusive, sexual, or synthetic by trusted child-safety attestors.
- Parents choose which attestors and lists to plug in and can lock Reality Switch settings behind device auth (FaceID/biometrics).
- The protocol makes this auditable: who signed what, when, and about which Packet.

### 3.2 Better evidence and coordination for real abuse

Strata's attestation model supports the actors formalized in CSAR (NGOs, hotlines, labs, media, platforms, the EU Centre):
- Publish signed claims about content ("this hash is confirmed CSAM and must be blocked", "this user is engaged in grooming behavior", "this image is miscaptioned/out-of-context").
- Publish signed corrections and retractions to unwind mistakes without losing history.
- Retroactive consensus lets clients say: if enough trusted child-safety attestors agree, blur or hide by default — without a single global oracle or hard delete.

### 3.3 Aligning with CSAR's risk-based duties

- **Risk assessment:** provenance + packet-level attestations form a rich, auditable risk signal.
- **Targeted measures:** high-risk features (open groups, public media sharing) can be bound to Strata-based controls while private encrypted chats stay private.
- **EU Centre integration:** the EU Centre can publish signed indicator sets as attestor outputs instead of opaque blacklists, so clients and platforms see why something is flagged and who stands behind the claim.

## 4. How Strata reduces surveillance risk

- **Relays are blind utilities** — they move Packets and see minimal metadata; they do not run classifiers or build feeds.
- **Filtering is client-side and user-controlled** — the Trust Engine runs on-device with the user's chosen attestors and settings; a child's phone can run Kids Mode without granting cloud providers scanning rights.
- **Evidence, not backdoors** — Strata relies on cryptographic provenance and signed third-party analysis; it does not weaken end-to-end encryption.
- **Transparency over secret models** — every attestation is a signed object with `attestor_id`, `subject`, `confidence`, and `method`; communities can pick different attestors without changing transport.

## 5. How this maps to concrete EU concerns

### For child-protection advocates
- Stronger tooling: signed provenance and edit-history trails; standard way for labs, NGOs, and the EU Centre to publish and retract findings.
- Avoids creating a single surveillance API that can be repurposed later.

### For privacy and digital-rights advocates
- Supports effective child-protection measures without normalizing message-scanning.
- Provides an open way to plug in child-safety signals locally instead of trusting closed platform models.

### For regulators and platforms
- Aligns with the Council's risk-based CSAR mandate and DSA systemic-risk duties without breaking encryption.
- Lets you ask, "does this service use verifiable provenance and attestor infrastructure?" instead of "did you build a proprietary scanning system?"
