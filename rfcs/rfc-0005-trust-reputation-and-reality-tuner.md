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
  "asset": "STRATA",
  "locked_until": 1718023000,
  "purpose": "reputation_boost",
  "created_at": 1715421500,
  "signature": "0xStakerSig..."
}
```
Possible formula:
`T_stake(i) = f( Σ sqrt(amount_by_distinct_stakers) )`, where:
- Multiple diverse stakers increase trust,
- Single whales have diminishing returns.

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

### 5.2 Trust Budgets
- Each identity has a trust budget per time window:
    - Can only issue strong trust edges to a limited number of new identities.
- Prevents a single identity from anointing a large number of bots quickly.

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
    
#### Standard (Default)
Recommended behavior:
- Show all Packets except:
    - Clear spam or abuse (based on reputation and attestations),
- Apply traffic‑light rings:
        - 🟢 strong provenance & trust,
        - 🟡 uncertain,
        - 🔴 synthetic/manipulated.
- Use `R_total` and provenance to influence ranking, not strict visibility.

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

## 8. Security & Privacy Considerations

- Reputation systems:
    - Can be gamed; algorithms must be monitored and iterated.
    - Should be transparent enough for community auditing.
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