# RFC-0005: Trust, Reputation, and Reality Tuner

- **Status:** Draft  
- **Author(s):** Strata Core Team  
- **Created:** 2025-11-25  
- **Updated:** 2025-11-25  

## 1. Summary

This RFC defines a **reference model** for:

- Computing identity trust and reputation,
- Mitigating Sybil attacks,
- The high‑level behavior of the **Reality Tuner** and **Reality Switch**.

It is **non‑normative**: clients MAY implement different algorithms while respecting the same data structures.

## 2. Motivation

Strata needs:

- A way to distinguish long‑lived, human‑backed identities from ephemeral bot swarms.
- A way to aggregate attestations and trust edges into usable signals.
- A simple UX (traffic lights + 3‑way toggle) over complex logic.

## 3. Inputs to Reputation

For an identity `i`, a client computes `R_total(i)` from:

1. **Seed Trust (`T_seed(i)`)**
   - Trust from designated seed gatekeepers (e.g. Strata Foundation, NGOs).
   - Captures initial bootstrap trust.

2. **Web-of-Trust (`T_wot(i)`)**
   - Derived from trust edges (RFC‑0001).
   - Personalized to the current user or community.

3. **Stake (`T_stake(i)`)**
   - Optional: value staked by various identities in favor of `i`.
   - Adds friction for Sybils.

4. **Egalitarian / Behavioral (`T_egal(i)`)**
   - Account age,
   - Consistent non‑abusive activity,
   - Absence of slashing or abuse reports.



## 4. Reference Trust Score

A reference formula:

```text
T(i) = w_seed * T_seed(i)
     + w_wot  * T_wot(i)
     + w_stake * T_stake(i)
     + w_egal * T_egal(i)

where w_seed + w_wot + w_stake + w_egal = 1
```

### 4.1 Seed Trust
- Seeds are StrataIDs configured in Bootstrap Documents.
- Seed distrust (e.g., external forks) is possible; clients MAY ignore specific seeds.
- Bootstrap Documents (whitepaper §10.1) **MUST** be:
  - Content‑addressed (`bootstrap_id = multihash(blake3-256(doc))`),
  - Signed by at least 3 distinct gatekeepers with different `diversity_tags` (e.g., region/org_type),
  - Versioned with `issued_at` and `expires_at` to avoid stale hijacks.
- Diversity tags **MUST** be anchored to verifiable evidence (e.g., DNS TXT for a domain, legal entity ID + jurisdiction, or verifiable credential). Clients count diversity by unique anchors, not just tag strings; if all anchors resolve to the same actor, clients **MUST** fail closed on diversity.
- Bootstrap Document contents **MUST** include:
  - `relays` list with relay DIDs and URLs,
  - `seed_gatekeepers` (StrataIDs used for `T_seed`),
  - `hardware_roots` and `model_roots` for genesis attestation chains.
- Clients **MUST** validate signatures against known gatekeeper keys and fail closed if signatures fall below threshold or diversity requirements.

`T_seed(i)` could be:
- The maximum or sum of trust edges from seed identities toward i, normalized.

### 4.2 Web‑of‑Trust
`T_wot(i)` is computed via:
- Personalized PageRank or similar graph algorithms,
- With:
  - Outgoing trust edges as edges in graph,
  - Decay by hop distance,
  - Caps on max influence from a single origin.

### 4.3 Stake
`T_stake(i)` is derived from stake entries:
```json
{
  "stake_id": "0xstake1",
  "staker": "did:strata:foundation",
  "beneficiary": "did:strata:journalists_guild",
  "amount": "1000.0",
  "asset": {
    "symbol": "STRATA",
    "chain": "solana-mainnet",          // or "offchain-escrow"
    "decimals": 9
  },
  "locked_until": 1718023000,
  "purpose": "reputation_boost",
  "created_at": 1715421500,
  "proof": {
    "type": "onchain-tx",
    "tx_id": "5Yg...Tx",
    "block_height": 123456,
    "proof_blob": "base64merkleproof..."
  },
  "signature": "0xStakerSig..."
}
```
Proof verification registry (minimum set):
- `chain = "solana-mainnet", proof.type = "onchain-tx"`: `tx_id` MUST reference a confirmed transaction; `proof_blob` MUST encode a Merkle/account-state proof binding `beneficiary`, `amount`, `locked_until`, and `asset` fields. Clients **MUST** validate inclusion against the referenced `block_height`.
- `chain = "ethereum-mainnet", proof.type = "onchain-tx"`: `proof_blob` MUST be an RLP/JSON receipt+Merkle proof to the canonical block header; same field bindings as above.
- `chain = "offchain-escrow", proof.type = "escrow-receipt"`: `proof_blob` MUST be a signed receipt by the escrow provider covering `stake_id`, `staker`, `beneficiary`, `amount`, `locked_until`, and an escrow attestor key declared in bootstrap or client policy.
Bootstrap documents MAY extend the registry with additional `chain` values and required proof codecs; clients **MUST** treat unknown chain/proof combinations as advisory only and weight them as zero.
Possible formula:
`T_stake(i) = f( Σ sqrt(amount_by_distinct_stakers) )`, where:
- Multiple diverse stakers increase trust,
- Single whales have diminishing returns.

Rules:
- `asset` MUST specify chain/context; unknown assets are ignored.
- `proof` MUST be provided; without a verifiable proof the stake entry is advisory only and should be down‑weighted to zero. Unknown `chain`/`proof.type` pairs follow the same rule.
- Slashing is expressed via Packets with `content.type = "STAKE_SLASH"` referencing `stake_id` and carrying evidence; slashing events **MUST** be signed by an authorized slasher identity declared in the original stake or by community policy.
- `locked_until` is enforced by verifying the underlying proof; clients **MUST** treat premature unlocks as slashable events.

### 4.4 Egalitarian / Behavioral
`T_egal(i)` may consider:
- `age` – time since first Packet from i,
- `activity` – number of non‑spam interactions,
- `report_score` – absence of abuse/spam flags.

Example:
```text
T_egal(i) = clamp( log(age_days + 1) / log(K) ) * (1 - report_penalty)
```
Where `K` is a scaling constant and `report_penalty` is derived from validated abuse reports.

## 5. Sybil Mitigation
Key techniques:

### 5.1 Age Gating
- New identities have capped influence for a grace period `G` (e.g. 30 days).
- During `G`, `T_wot(i)` and `T_stake(i)` contributions are limited.
- Age is measured using the earliest relay‑observed `received_at` for Packets from the identity, not author‑supplied timestamps.

### 5.2 Trust Budgets
- Each identity has a trust budget per time window:
    - Can only issue strong trust edges to a limited number of new identities.
- Prevents a single identity from anointing a large number of bots quickly.
- Reference parameters (implementations MAY tune):
  - `B_high` = 20 edges with `strength > 0.5` per 24h (rolling) per identity.
  - `B_low` = 100 edges with `strength <= 0.5` per 24h.
- Rolling-window accounting (using relay‑observed `received_at`, tie by packet_id):
```text
window = 24h
bucket(edge) = strength > 0.5 ? HIGH : LOW

on_issue(edge):
  purge(events where received_at < now - window)
  if count(issues in bucket) >= B_bucket:
     reject or emit with weight=0
  else:
     record issue event in bucket

on_revocation(edge):
  purge(events where received_at < now - window)
  record revocation event (does not delete issue events)
  // budget only frees when the original issue event ages out of window
```
- Budgets for HIGH and LOW are independent; revoking a LOW edge does not affect HIGH capacity and vice versa.
- Multiple revocations inside the window do not accelerate refunds; the original issue’s timestamp controls when budget is restored.
- Relays **SHOULD** enforce budgets at ingest; clients **MUST** down‑weight edges beyond budget to zero to remain interoperable.

### 5.3 Cluster Detection
- Identify dense subgraphs with high internal trust but low external edges.
- Down‑weight trust contributions from such clusters until bridged by external, high‑trust nodes.

## 6. Reality Tuner

### 6.1 Inputs

For each Packet `P`, Reality Tuner considers:
- `author_id` and `R_total(author_id)`,
- `provenance_header.origin_type`,
- Attestations and retroactive consensus status (RFC‑0003),
- User’s Reality Switch mode,
- User’s attestor and seed preferences.

### 6.2 Outputs
- Feed inclusion (show / hide),
- Visibility level (normal / blurred / gated),
- Label:
    - Provenance color (🟢/🟡/🔴),
    - Warnings (e.g., “Manipulated,” “Likely synthetic”).

Clients SHOULD treat the Reality Tuner as the canonical decision function for feed inclusion and visibility:
- The Reality Switch (Strict / Standard / Wild) is an input to the Tuner configuration, not a separate filtering layer.
- Application code that assembles feeds SHOULD NOT implement independent “mode flags” that bypass the Tuner’s show / blur / hide decisions, except for purely UX‑driven constraints (e.g., different surfaces over the same underlying Tuner output).
- When an application wants a stricter or looser feed profile, it SHOULD do so by changing Tuner parameters (e.g., thresholds, quorum weights) rather than bolting on additional opaque filters.

### 6.3 Reality Switch Modes

#### Strict (Grandma Mode)
Recommended behavior:
- Show only Packets where:
    - `R_total(author) >= T_strict_min`,
    - Provenance is not UNKNOWN unless author is highly trusted,
    - No retroactive consensus for “MANIPULATED” or “SPAM.”
- For red‑ring content:
    - Hide or blur by default,
    - Require explicit user action to view.
- When retroactive consensus quorum is reached for negative claims (e.g., `MANIPULATED`, `ORIGIN_LIKELY_SYNTH`, `SPAM`, `ABUSIVE` per RFC‑0003 §7), the Reality Tuner SHOULD hide or heavily blur the corresponding Packets by default in Strict mode, and require explicit user action to reveal them if they are shown at all.
    
#### Standard (Default)
Recommended behavior:
- Show all Packets except:
    - Clear spam or abuse (based on reputation and attestations),
- Apply traffic‑light rings:
        - 🟢 strong provenance & trust,
        - 🟡 uncertain,
        - 🔴 synthetic/manipulated.
- Use `R_total` and provenance to influence ranking, not strict visibility.
- When negative quorum is reached, the Tuner SHOULD blur thumbnails and apply red‑ring + warning labels by default and down‑rank the content in feeds, rather than relying on a separate “flagged” mode bit outside the Tuner.

#### Wild (Developer Mode)
Recommended behavior:
- Show nearly all Packets except obvious DoS/garbage.
- Surface provenance and attestation details more prominently.
- Allow user to “opt into” seeing even heavily flagged content.

## 7. Traffic‑Light Mapping

Reference mapping from signals to ring color:
- **Green (🟢)** if:
    - `origin_type = HARDWARE_SECURE_ENCLAVE` AND
    - `R_total(author) >= T_green_min`, OR
    - Quorum of trusted attestations asserts `ORIGIN_LIKELY_HUMAN` with high confidence.
- **Red (🔴)** if:
    - `origin_type = AI_MODEL`, OR
    - Quorum asserts `MANIPULATED` or `ORIGIN_LIKELY_SYNTH` with high confidence.
- **Yellow (🟡)** otherwise:
    - `origin_type = UNKNOWN` or `SOFTWARE` with no strong attestations,
    - Mixed or conflicting attestations.
    
Thresholds (`T_green_min`, etc.) are implementation‑specific.

Clients SHOULD derive both ring color and visibility (show / blur / hide) from the Reality Tuner’s evaluation of provenance, `R_total`, and attestation quorum status, rather than from ad‑hoc per‑feature flags.

## 8. Security & Privacy Considerations

- Reputation systems:
    - Can be gamed; algorithms must be monitored and iterated.
    - Should be transparent enough for community auditing.
- Seed abuse:
    - Bootstrap Documents must meet signature/diversity thresholds; clients should surface provenance of seeds to users.
- Stake forgery:
    - Stake entries without verifiable proofs or slashing rules MUST be treated as zero weight.
- Privacy:
    - Publishing trust edges reveals social connections.
    - Clients SHOULD offer options for private/local trust edges not shared publicly.

## 9. Backwards Compatibility
- Reputation algorithms may change; underlying data structures (trust edges, stakes) MUST remain stable.
- Clients MAY version their Reality Tuner configurations and provide migration paths.

## 10. Reference Implementation Notes
- Reference client **SHOULD**:
    - Cache `R_total` for identities seen in recent feeds,
    - Recompute in background when new trust edges or stakes appear,
    - Allow users to inspect “Why is this marked red?” with a breakdown of signals.
