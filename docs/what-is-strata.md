# What Is Strata?

Strata is a decentralized verification layer for the internet. It is designed to defend:

- Reality — against synthetic media, deepfakes, and bot swarms.
- Liberty — against centralized surveillance, mandated backdoors, and platform-level censorship.

Instead of asking “what does the platform say is true?”, Strata asks: **who signed this, where did it come from, and what does my local policy say I should do with it?**

## Core Idea

Strata moves from platform moderation to user sovereignty:

- Every unit of information is a **Data Packet**: content, cryptographic provenance, optional attestations, signature, and audit trail.
- Servers become **dumb relays**: they store and forward encrypted packets but do not assemble feeds or decide what is allowed.
- Users run a local **Trust Engine** (Reality Switch presets) on their own device to decide what to see, blur, or hide using their own trust graph and settings.

## Problems Strata Targets

1. **Synthetic media flood** — realistic fake videos, audio, images, and bot-driven narratives make intuition unreliable.
2. **Legislative overreach and platform chokepoints** — pressure for client-side scanning, “lawful access,” and top-down moderation through app stores and cloud hosts.

Strata offers a path based on cryptographic provenance plus local control.

## High-Level Design

Strata has four conceptual layers:

1. **Identity & Trust (Layer 0)** — StrataIDs (DIDs) backed by keys; trust edges form a Web-of-Trust.
2. **Provenance (Layer 1)** — Packets include a `provenance_header` and may reference Genesis Events describing how media was created and edited.
3. **Transport Mesh (Layer 2)** — relays store and forward encrypted Packets; content is addressed by hash; no single relay is required.
4. **Applications & Trust Engine (Layer 3)** — clients implement feeds plus the local policy engine (Reality Switch) to decide what to show, blur, or hide.

## UX Philosophy: “Invisible Crypto”

- Onboarding: install → create account → FaceID/TouchID/Passkey → done. Keys live in secure hardware; no seed phrase prompts in the main flow.
- Provenance: traffic-light rings on posts (🟢 strong provenance, 🟡 unknown/legacy, 🔴 synthetic/manipulated per trusted attestations).
- Control: a **Reality Switch** with three presets — Strict, Standard, Wild — backing the Trust Engine configuration.

## What Strata Is Not

- Not a truth oracle — it exposes provenance and signed opinions; it does not declare truth.
- Not a censoring platform — no global bans or takedowns in the protocol; relays and users pick their own policies.
- Not a walled garden — open RFCs any app or relay can adopt.

## Where to Go Next

- Protocol details: see the whitepaper and RFCs in this repo.
- Architecture overview: see `docs/how-strata-helps-protect.md` for an applied example around child protection and privacy.
