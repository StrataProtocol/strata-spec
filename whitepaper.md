<!-- whitepaper.md -->

# The Strata Protocol  
## Cryptographic Provenance and Client‑Side Sovereignty

---

## 1. Abstract

The Strata Protocol is a decentralized verification layer for the internet. It is designed to defend:

- **Epistemic reality** – against synthetic media, deepfakes, and bot swarms.
- **Civil liberty** – against centralized surveillance, backdoors, and platform‑centric censorship.

Strata shifts from **platform moderation** to **user sovereignty**:

- Every unit of information is a **Data Packet**: content + cryptographic provenance + attestations + audit trail.
- Servers act as **dumb relays**, forwarding opaque, encrypted packets.
- Each user runs a **Reality Tuner** on their own device that decides what to show, blur, or ignore.

Identity is cryptographic (keypairs and DIDs), not administrative (database rows and KYC). Provenance is explicit: where possible, hardware‑signed capture and model‑stamped generation; otherwise, clearly marked as unknown. Moderation is local: the network does not delete, it only labels; users and clients choose which labels to believe.

Strata is designed to be **invisible** to everyday users. They see a simple social app with a traffic‑light trust indicator and a three‑way “Reality Switch.” All the cryptography happens under the hood.

---

## 2. Background and Problem Statement

The internet is facing a two‑front crisis:

### 2.1 Synthetic Media & Deepfakes

- Generative models can fabricate highly realistic images, videos, and audio.
- Social feeds and messaging apps are vulnerable to mass‑produced unreality:
  - Fake speeches,
  - Staged “evidence,”
  - Synthetic personas.

Traditional detection approaches are an arms race: every improved classifier is followed by improved generators that evade it. “Is this real?” is no longer answerable by human intuition alone.

### 2.2 Legislative Overreach & Platform Centralization

At the same time, governments and regulators are:

- Proposing or enacting **client‑side scanning** of private messages.
- Demanding **“lawful access”** backdoors in encrypted systems.
- Imposing **proactive moderation obligations** on platforms.

Centralized platforms and app stores become chokepoints:

- App stores can delist non‑compliant apps.
- Cloud providers can shut down servers.
- One‑size‑fits‑all moderation policies are enforced top‑down.

The result:

- Epistemic trust is collapsing (we don’t know what to believe).
- Individual autonomy is shrinking (we don’t control what we see).
- Infrastructure is fragile (a few choke points can silence many voices).

---

## 3. Design Goals

Strata’s design is driven by the following goals:

1. **User Sovereignty**
   - Users choose what to trust and what to see.
   - The protocol does not mandate any global “correct” feed.

2. **Provenance over Guesswork**
   - Prefer cryptographic proofs (signatures, hardware attestations) over opaque detection models.
   - Where detection is used, treat it as signed opinion, not truth.

3. **Censorship Resistance by Architecture**
   - No global delete in the protocol.
   - Relays are interchangeable; clients are forkable; bootstrap is multi‑source.
   - The default design makes centralized, silent feed manipulation difficult.

4. **Legal Plausibility for Deployments**
   - The protocol itself is neutral and open.
   - Individual relays and clients can adopt local compliance policies (e.g. blocking known illegal hashes) without weakening the protocol.

5. **Grandma‑Compatible UX**
   - Onboarding has to feel like TikTok/Instagram:
     - Install → “Create Account” → FaceID/TouchID → done.
   - No seed phrases, no “connect wallet” screens in the primary flow.
   - Cryptographic concepts are hidden behind familiar metaphors (traffic lights, toggles).

6. **Interoperability & Extensibility**
   - Use well‑known primitives and standards where possible (DIDs, content addressing, hardware signing).
   - Make it easy for third‑party clients, relays, and attestors to plug in.

---

## 4. Strata Axioms

These axioms are the philosophical core of Strata.

### 4.1 Identity is Cryptographic, Not Administrative

- You are a keypair (one or more), not a row in a platform’s database.
- Your **StrataID** is a DID derived from cryptographic keys.
- Reputation and trust travel with your StrataID across apps and relays.
- Government ID/KYC is not required at the protocol level.

### 4.2 Provenance First, Attestations Second, Detection Last

- When available, **hardware capture** and **model‑stamped generation** provide strong provenance.
- Forensic analysis and AI detection are modeled as **attestations**:
  - “This looks synthetic.”
  - “This matches on‑camera capture.”
- Attestations are signed claims from many actors (labs, media, NGOs, clients).
- Users choose which attestors they trust; no single “oracle of truth.”

### 4.3 Client‑Side Sovereignty (The Legal Shield)

- Relays provide raw, encrypted packet streams.
- The **Reality Tuner** on the user’s device decides:
  - Which packets to accept into the feed,
  - How to label or blur them.
- Because filtering is done on‑device, relays behave like utilities, not publishers.

### 4.4 Data Packets, Not Posts

- A “post” is just content.
- A **Packet** is content + metadata + provenance + attestations + signature.
- Packets are tamper‑evident and auditable.

### 4.5 Censorship Resistance by Architecture

- The protocol has no global delete operation.
- Relays may choose not to store or forward certain packets; users can switch to others.
- Cryptographic identity and content addressing make it easy for clients to connect to new relays and rediscover content.

### 4.6 The Grandma Axiom (Radical Simplicity)

- If a user must understand “private keys,” “relays,” or “DIDs” to use Strata, Strata has failed.
- Security defaults must be safe; advanced options must be discoverable, not required.
- Complexity lives in the engine room, not in the driver’s seat.

### 4.7 Many Gatekeepers, No Single Gate

- Discovery uses **Bootstrap Documents** signed by multiple introducers.
- Anyone can host a mirror, publish their own bootstrap documents, or define their own gatekeepers.
- There is no single canonical hostname or entity that users must trust.

### 4.8 Trust Evolves Over Time

- In early stages, seed stakeholders and stake help bootstrap basic Sybil resistance.
- Over time, weight shifts to:
  - Web‑of‑Trust,
  - Account age,
  - Long‑term behavior.
- The reputation model is explicitly designed to be adjustable.

---

## 5. Architecture Overview

Strata is a four‑layer architecture:

1. **Layer 0: Identity & Trust**
   - StrataIDs (DIDs) and trust relationships between them.
2. **Layer 1: Provenance Ledger**
   - Genesis events and provenance headers for media.
3. **Layer 2: Transport Mesh**
   - Relays and content‑addressed storage.
4. **Layer 3: Applications & Reality Tuner**
   - Clients, UX, Reality Switch, filters.

### 5.1 Layer 0 – Identity & Trust (Nodes)

- Each user controls one or more **StrataIDs**, represented as DIDs.
- Keys are managed locally, preferably in secure enclaves or via passkeys.
- Trust is expressed via signed edges:
  - “I vouch for this identity as human.”
  - “I trust this organization as a reliable attestor.”
- Reputation is composite:
  - Social trust graph,
  - Optional stake,
  - Age and behavior.

### 5.2 Layer 1 – Provenance Ledger (Chain of Custody)

For media content, Strata records:

- A **Genesis Event**:
  - Type A: Hardware‑signed capture.
  - Type B: AI‑generated (model/provider stamps metadata).
  - Type C: Unknown or edited/legacy.
- A **provenance header** attached to Packets:
  - References the genesis media hash,
  - Records edit history,
  - Marks origin_type.

Hardware‑signed and model‑stamped content can later be validated against issuer keys; unknown content is clearly marked.

### 5.3 Layer 2 – Transport Mesh (Relays)

Relays are minimal, content‑agnostic servers:

- Accept Packets from clients.
- Store them.
- Forward them to subscribers.
- Content is referenced via hashes (e.g. IPFS/Arweave).

Relays may apply local policies (blocking some keys or hashes) but do not interpret decrypted content. Multiple relays can exist in parallel; users and clients choose which to connect to.

### 5.4 Layer 3 – Applications & Reality Tuner (Clients)

Clients implement:

- UX (feed, profiles, composition),
- Key generation and identity handling,
- Reality Tuner policies.

**Official Strata Apps** are:

- Open‑source PWAs (with optional native wrappers),
- Implement the core UX ideas:
  - TikTok‑style onboarding,
  - Traffic‑light provenance rings,
  - 3‑position Reality Switch.

Any third‑party app can implement the Strata protocol; “official” apps simply follow certain design and governance principles.

---

## 6. Identity & Trust (Layer 0)

### 6.1 StrataID – Identity as a DID

- Format: `did:strata:<id>`.
- `<id>` is derived from public keys using standard multibase/multicodec encoding.
- A DID Document includes:
  - Verification methods (public keys),
  - Authentication methods,
  - Optional services (preferred relays, etc.).

Users may have:

- **Public identity** – long‑lived, reputation‑bearing.
- **Pseudonymous identity** – used where anonymity is desired.
- **Ephemeral identity** – throwaway/burner keys.

### 6.2 Key Management & Recovery

- Default onboarding:
  - “Create account” → device generates keys in secure storage.
  - FaceID/TouchID/Passkey unlock; no manual key handling.
- Advanced users can export, rotate, and manage keys manually.

Recovery options may include:

- Encrypted backups (user‑controlled storage),
- Social recovery (trusted contacts),
- Hardware tokens.

### 6.3 Trust Links

Trust is modeled through **trust edges**:

- Alice (DID A) can issue a signed trust edge about Bob (DID B):
  - Strength (0–1),
  - Context (friend, colleague, source, etc.).

These edges form a graph. Clients can compute personalized trust scores based on:

- Direct links,
- Indirect links (hops),
- Age of trust relationships.

### 6.4 Composite Reputation

To classify identities (e.g., “likely human, long‑lived, non‑spammy”), Strata uses a composite reputation score \(R(i)\):

- **R_social** – web‑of‑trust score as seen by the current user.
- **R_stake** – value staked on an identity’s behavior by diverse stakeholders.
- **R_egal** – egalitarian component based on age, consistent non‑abusive activity.

Clients can implement:

> R_total(i) = α·R_social(i) + β·R_stake(i) + γ·R_egal(i), with α+β+γ = 1.

Weights can evolve over time (e.g. β smaller as network matures).

---

## 7. Provenance & Attestations (Layer 1)

### 7.1 Genesis Events

Every media asset in Strata has at least one **Genesis Event**:

- **Type A — Hardware‑Signed Capture**
  - Device (camera/phone) signs the raw media hash and capture metadata.
  - Provides strong evidence the media was captured by a specific class of device.

- **Type B — AI‑Generated**
  - A cooperating model provider stamps generation metadata and signs it:
    - Model family,
    - Version,
    - Prompt hash (optional),
    - Generation settings.

- **Type C — Unknown/Edited/Legacy**
  - No hardware/model signature or chain has been broken by editing.
  - Marked as `UNKNOWN` or `SOFTWARE`.

Genesis Events are immutable claims about media hashes. Edits append to the history, they do not rewrite genesis.

### 7.2 Provenance Headers

Packets referencing media include a `provenance_header`:

- `origin_type` (HARDWARE_SECURE_ENCLAVE, AI_MODEL, SOFTWARE, UNKNOWN),
- `genesis_media_hash`,
- `device_id` and `device_signature` (for hardware‑signed media),
- `edit_history` (list of transforms with timestamps and signatures).

This header lets clients and attestors reconstruct a content’s chain of custody.

### 7.3 Attestations: Multiple Eyes on the Same Content

Provenance alone does not guarantee truth. Strata supports **attestations**:

- Signed claims about a packet, such as:
  - Likely on‑camera capture,
  - Likely synthetic,
  - Manipulated (e.g., spliced, resampled),
  - Misleading out of context.

Attestors can be:

- Official Strata client(s),
- Third‑party forensic labs,
- Fact‑checking NGOs,
- Journalists’ collectives,
- Model providers,
- Independent experts.

Users/clients:

- Choose which attestors to trust,
- Decide how to weigh conflicting claims,
- Maintain their own attestor allow/deny lists.

Attestations are first‑class objects in the protocol and can be combined to form **retroactive consensus**.

### 7.4 Traffic‑Light UX

Complex provenance and attestation data is abstracted into a simple visual language:

- 🟢 **Green ring** – strong provenance and/or strongly positive attestation from trusted sources:
  - Hardware‑signed capture, or
  - Multiple high‑trust attestations asserting “likely human capture / unmanipulated.”
- 🟡 **Yellow ring** – uncertain or legacy:
  - Origin unknown or edited,
  - Mixed or insufficient attestations.
- 🔴 **Red ring** – synthetic or manipulated according to trusted attestors:
  - AI_MODEL origin, or
  - Quorum of trusted attestations asserting manipulation/synthetic.

Green does not mean “true”; red does not mean “false opinion.” Rings are about **provenance and evidence**, not ideology.

---

## 8. Transport Mesh & “Magic Relays” (Layer 2)

Relays are the network’s plumbing:

- **Opaque transport**:
  - Relays see metadata and encrypted blobs; they do not interpret decrypted content.
- **Content‑addressed storage**:
  - Media is stored by hash, not by URL or UID.
- **Stateless routing** (conceptually):
  - Clients can connect to one or many relays; relays can peer with one another.

### 8.1 Magic Relays UX

Users do **not** manually:

- Pick servers,
- Configure peers,
- Understand relay policies.

Clients:

- Auto‑discover and connect to a set of relays based on bootstrap documents,
- Fail over to others when one is unreachable or hostile,
- Hide all of this behind the familiar concept of “being online.”

---

## 9. Applications, Reality Tuner & UX (Layer 3)

### 9.1 TikTok‑Style Onboarding (“Invisible Crypto”)

Onboarding flow for official Strata apps:

1. Install app / open PWA.
2. Tap “Create Account.”
3. Device prompts for FaceID/TouchID/Passkey approval.
4. Account is ready.

What happens behind the scenes:

- Keypair generated in secure enclave / WebAuthn.
- StrataID (DID) derived from public key.
- Optional local backup mechanisms initialized.

What does **not** happen by default:

- No visible “generating keys” spinner.
- No “connect wallet” or “backup these 24 words” modal.
- No immediate introduction of jargon.

Advanced settings allow power‑users to see and export keys.

### 9.2 Reality Switch

The **Reality Switch** is the primary high‑level control:

1. **Strict (Grandma Mode)**
   - Prioritize:
     - Green‑ring content,
     - Authors with high R_total (trusted).
   - Blur or hide:
     - Red‑ring content by default.
   - Behaves more cautiously around unknown provenance and low‑reputation actors.

2. **Standard (Default)**
   - Show a broad mix of content:
     - All colors appear, but clearly labeled.
   - Use provenance and reputation to order/annotate, not fully hide.

3. **Wild (Developer Mode)**
   - Show the raw firehose, with minimal filtering beyond spam/DoS.
   - Useful for researchers, developers, and those who want maximum visibility.

Clients may allow advanced users to define custom profiles and policies beyond these three presets.

### 9.3 Reality Tuner

Under the hood, the Reality Tuner:

- Consumes:
  - Packets,
  - Provenance headers,
  - Attestations,
  - Reputation scores,
  - User policy and attestor preferences.
- Produces:
  - A ranked, filtered feed,
  - Per‑packet labels (rings, warnings, blur/hide decisions).

The Tuner logic is client‑specific but guided by protocol‑defined data structures and reference algorithms.

---

## 10. Discovery & Gatekeepers

To avoid dependence on a single domain (e.g. `strata.xyz`), Strata uses:

### 10.1 Bootstrap Documents

- Signed JSON documents listing:
  - Initial relays,
  - Seed gatekeepers/introducers,
  - Basic metadata.
- Hosted on arbitrary URLs; mirrored widely.

### 10.2 Gatekeepers / Introducers

- Gatekeepers are StrataIDs (organizations or communities) that:
  - Publish bootstrap documents,
  - Provide initial trust anchors.
- Examples:
  - Strata Foundation,
  - Press freedom NGOs,
  - Regional community orgs.

Clients:

- Ship with multiple bootstrap URLs from diverse gatekeepers.
- Let users add/replace gatekeepers via URL, QR code, or DID input.

“Official” bootstrap sets are just one of many possible options.

---

## 11. Retroactive Consensus (Anti‑Viral Mechanism)

Strata’s **Retroactive Consensus** mechanism addresses the case where:

- A piece of content (e.g. a video) goes viral as implicitly “real,”
- Later forensic work reveals it is synthetic or manipulated.

Rather than global deletion, Strata enables:

1. **Attestors** (labs, NGOs, media) issue signed attestations about the content:
   - E.g. verdict `MANIPULATED` with high confidence.
2. **Quorum logic** on the client side:
   - If enough high‑reputation attestors a user trusts agree,
   - The client treats the content as falsified.
3. The Reality Tuner:
   - Blurs the thumbnail by default,
   - Adds warnings and red‑ring labels,
   - Optionally hides it in Strict mode.

Characteristics:

- Retroactive consensus is **evidence‑driven**, not opinion‑driven.
- The original content is not deleted at the protocol level.
- Different clients or user profiles may disagree on when quorum has been reached.

This mechanism is designed to curb viral deepfakes without becoming a centralized, political censorship tool.

---

## 12. Threat Model & Limitations

### 12.1 Threats Considered

- **Synthetic Media Generators**
  - Mass production of deepfakes and synthetic narratives.
- **Sybil Swarms**
  - Large botnets creating mutual trust to appear “human.”
- **State‑Level Censors**
  - Governments pressuring relays, client developers, and app stores.
- **Abuse, Spam, and Harassment**
  - Malicious behavior by users exploiting openness.

### 12.2 Limitations

Strata does **not**:

- Prevent OS vendors from implementing client‑side scanning outside its control.
- Prevent governments from banning specific apps or relays in their jurisdiction.
- Guarantee “truth” in any philosophical sense.
- Solve all abuse challenges (e.g. harassment still requires social/legal responses).

Strata **does**:

- Provide a structural foundation where provenance is explicit,
- Make it easier for users to choose their trust relationships,
- Reduce the power of single platforms as arbiters of speech.

---

## 13. Ecosystem & Governance

Key ecosystem roles:

- **Strata Foundation**
  - Maintains reference clients and relays,
  - Curates some Bootstrap Documents,
  - Coordinates hardware/model integrations.
- **Relay Operators**
  - Run infrastructure,
  - Choose local policies (within their legal/ethical frameworks).
- **Attestors**
  - Create attestations and forensic analyses.
- **Client Developers**
  - Build apps tailored to different communities and preferences.
- **Communities**
  - Maintain their own gatekeepers, filters, and reputational norms.

Governance principles:

- Open, forkable code and specifications,
- Transparent bootstrap and gatekeeping logic,
- Avoidance of hidden centralization.

---

## 14. Implementation Roadmap

### Phase 1 – Identity & Text Packets (“Cold Node”)

- Implement StrataID creation (local keys).
- Text‑only Packets with signatures.
- Minimal relay with WebSocket pub/sub.
- Simple React/Vite PWA client.

### Phase 2 – Provenance & Attestations

- Implement provenance_header and Genesis Events.
- Integrate at least one hardware signing path.
- Define attestation schema; have the official client issue simple attestations.

### Phase 3 – Trust & Reputation v0

- Implement trust edges and basic trust graph.
- Introduce simple reputation scoring (R_total).
- Map R_total + provenance into traffic‑light labels.

### Phase 4 – Discovery & Multiple Gatekeepers

- Finalize Bootstrap Document format.
- Ship multiple bootstrap sets.
- UI for adding/replacing gatekeepers.

### Phase 5 – Advanced Features

- Optional onion‑style routing / metadata minimization.
- Richer Sybil detection and cluster analysis.
- Ecosystem of independent attestors, clients, and relays.

---

## 15. Conclusion

Strata is not a truth oracle. It is a **substrate** for building healthier information ecosystems:

- Identities backed by cryptography instead of platform accounts.
- Content traced by provenance and attestations instead of opaque moderation.
- User devices making final decisions instead of centralized feeds.

In a future where generative models can flood the world with convincing fakes and centralized infrastructure is under pressure to surveil and censor, Strata offers a third path: cryptographic provenance, pluralistic verification, and client‑side sovereignty.
