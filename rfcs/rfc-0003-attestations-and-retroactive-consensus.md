<!-- rfcs/rfc-0003-attestations-and-retroactive-consensus.md -->

# RFC-0003: Attestations and Retroactive Consensus

- **Status:** Draft  
- **Author(s):** Strata Core Team  
- **Created:** 2025-11-25  
- **Updated:** 2025-11-25  

## 1. Summary

This RFC defines:

- The **Attestation** object for making signed claims about Packets,
- The types of claims (synthetic, manipulated, etc.),
- How clients aggregate and interpret attestations,
- A model for **Retroactive Consensus** (anti‑viral mechanism).

## 2. Motivation

Even with strong provenance, we still need:

- Forensic analysis (e.g., detecting manipulation),
- Fact‑checking (e.g., identifying miscaptioned videos),
- AI detection (as one input, not as an oracle).

These must be:

- Explicit,
- Signed,
- Pluralistic (many attestors, not one).

Retroactive consensus leverages attestations to downgrade or blur widely‑shared content when sufficient evidence of falsification emerges.

## 3. Attestation Structure

### 3.1 Embedded Attestation in Packet

Packets MAY include embedded attestations in the `attestations` array:

```jsonc
"attestations": [
  {
    "attestation_id": "0xatt1",
    "attestor_id": "did:strata:official_client",
    "attestor_type": "CLIENT",      // CLIENT | NGO | LAB | MEDIA | MODEL_PROVIDER | OTHER
    "subject": "ORIGIN_LIKELY_HUMAN",   // see below
    "confidence": 0.87,             // 0–1
    "method": "ML_MODEL_V1",
    "issued_at": 1715421250,
    "metadata": {},
    "signature": "0xAttestSig..."
  }
]
```

Fields:
- `attestation_id`: unique identifier for this attestation.
- `attestor_id`: StrataID of the attestor.
- `attestor_type`: classification (for UX).
- `subject`: claim subject (e.g., ORIGIN_LIKELY_HUMAN).
- `confidence`: numeric confidence in [0,1].
- `method`: free‑form description or version string.
- `issued_at`: timestamp.
- `metadata`: optional structured data.
- `signature`: attestor’s signature over the attestation payload.

### 3.2 Standalone Attestation Packet

Attestations MAY also be sent as standalone Packets (`content.type = "ATTESTATION"`), referencing a `target_packet`:

```jsonc
{
  "packet_id": "0xattpkt2",
  "version": 1,
  "timestamp": 1715422200,
  "author_id": "did:ngo:forensic_lab",

  "content": {
    "type": "ATTESTATION",
    "target_packet": "0x8f2...a1",
    "attestation": {
      "attestation_id": "0xatt2",
      "attestor_id": "did:ngo:forensic_lab",
      "attestor_type": "LAB",
      "subject": "MANIPULATED",
      "confidence": 0.95,
      "method": "FORENSIC_SUITE_Y",
      "issued_at": 1715422200,
      "metadata": {
        "note": "Resampled regions, lighting inconsistencies..."
      },
      "signature": "0xLabSig..."
    }
  },

  "signature": "0xFinalPacketSignature..."
}
```

Clients SHOULD:
- Index standalone attestations by `target_packet`,
- Merge embedded and external attestations for decision making.

## 4. Claim Types

The `subject` field identifies what is being claimed. Recommended initial enumeration:

- Provenance Analysis:
    - `ORIGIN_LIKELY_HUMAN`
    - `ORIGIN_LIKELY_SYNTH`
    - `MANIPULATED`
    - `UNALTERED_HARDWARE_CAPTURE`

- Fact-Checking:
    - `CAPTION_MISLEADING`
    - `OUT_OF_CONTEXT`
    - `FACTUAL_INACCURACY`
    - `MISATTRIBUTED_SOURCE`

- Spam/Abuse:
    - `SPAM`
    - `ABUSIVE`
    - `SCAM`

Each claim type may have its own thresholds and handling in clients.

## 5. Attestor Types

`attestor_type` is a short label for the kind of attestor:

- `CLIENT` – Strata client software (e.g., doing local AI detection).
- `NGO` – non‑governmental organization (e.g., fact‑checking NGO).
- `LAB` – forensic analysis lab.
- `MEDIA` – newsroom or journalistic collective.
- `MODEL_PROVIDER` – AI model or platform that generated content.
- `OTHER` – fallback.

This field is primarily for UX; trust is determined by attestor_id and reputation, not by attestor_type.

## 6. Trust in Attestations

Clients maintain **attestor policies**:

- Which `attestor_ids` are trusted,
- Whether specific claim types are accepted from them,
- How much weight each attestor’s claims carry.

For example, a user may:
- Trust only `LAB` and `NGO` attestors for `MANIPULATED` claims,
- Trust `MODEL_PROVIDER` attestations about content they generated,
- Ignore all `CLIENT` attestations.

Attestor reputation can itself be computed using the same trust and stake mechanisms as any other identity.

## 7. Retroactive Consensus

### 7.1 Problem

Content often goes viral before forensic analysis finishes. Later, strong evidence may show:

- The video is a deepfake,
- The audio is spliced,
- The clip is miscaptioned and misleading.

We want clients to be able to retroactively downgrade such content without:
- Deleting it from the protocol,
- Enforcing a single global verdict.

### 7.2 Quorum Model

Let:

- `P` = target packet,
- `A_support` = set of attestations about `P` supporting a particular claim `C` (e.g., `MANIPULATED`),
- `R_total(i)` = reputation score of attestor `i` (see RFC‑0005).

A client defines a **quorum** for claim `C` when:

1. There are at least `N_min` distinct attestors in `A_support`,
2. The sum of their reputation scores exceeds `W_min`:
    - `Σ_{i in A_support} R_total(i) ≥ W_min`,
3. Attestors belong to at least `C_min` distinct trust clusters (not all from the same tight community),
4. The oldest supporting attestation is at least `T_min` seconds old (to resist flash‑mobs).

Values (`N_min`, `W_min`, `C_min`, `T_min`) are client‑configurable and may differ:
- Per claim type,
- Per Reality Switch mode (Strict vs Standard vs Wild).

### 7.3 Client Behavior on Quorum

When quorum is reached for a claim like `MANIPULATED` or `ORIGIN_LIKELY_SYNTH`:

- **Strict mode**: 
    - Hide or heavily blur the content by default,
    - Show prominent warnings if user attempts to view.

- **Standard mode**: 
    - Blur thumbnails by default,
    - Show clear warnings and red rings.

- **Wild mode**: 
    - Show content but label with warnings and provenance details.

Clients **MUST NOT** be required by the protocol to delete or permanently block P. Retroactive consensus is advisory to the Reality Tuner.

### 8. Security & Abuse Considerations

- **Attestor compromise:**
    - If a trusted attestor’s keys are compromised, bad actors could issue false claims.
    - Clients **SHOULD** support revocation and rapid de‑trusting of attestors.

- **False consensus attacks:**
    - Sybil clusters could try to form fake quorum.
    - Reputation and cluster analysis (RFC‑0005) are critical to mitigate this.

- **Privacy:**
    - Attestations may reveal that a user/system analyzed certain content.
    - Sensitive attestors **MAY** choose to publish only aggregated or anonymized attestations.

### 9. Backwards Compatibility
- Introduction of new claim types or attestor types MUST be backward compatible.
- Unknown `subject` or `attestor_type` values MUST be safely ignored by clients, treated as unrecognized but not necessarily untrusted.

### 10. Reference Implementation Notes

Reference client **SHOULD**:
- Cache attestations per `target_packet`,
- Allow user to inspect all attestations for a given Packet,
- Expose settings to choose which attestors to trust.
