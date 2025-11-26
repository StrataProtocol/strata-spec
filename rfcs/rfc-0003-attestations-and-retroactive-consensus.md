<!-- rfcs/rfc-0003-attestations-and-retroactive-consensus.md -->

# RFC-0003: Attestations and Retroactive Consensus

- **Status:** Draft
- **Author(s):** Strata Core Team
- **Created:** 2025-11-25
- **Updated:** 2025-11-26

> **Note:** `packet_id` and content-addressed IDs use `0x` + lowercase hex encoding. See RFC-0000 5.2-5.4.  

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

Packets MAY include embedded attestations in the top-level `attestations` array (see RFC-0002 §3.1):

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
  "packet_id": "0x1e20c4d5e6f7081920a1b2c3d4e5f6078192a3b4c5d6e7f8091a2b3c4d5e6f7082",
  "version": 1,
  "timestamp": 1715422200,
  "author_id": "did:ngo:forensic_lab",

    "content": {
      "type": "ATTESTATION",
      "target_packet": "0x1e208f2c3a4b5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f7a1",
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
- Before an attestation contributes to any Reality Tuner, quorum, or reputation logic, clients **MUST**:
  - Validate the containing Packet per RFC-0002 (packet signature, `author_id`, and time windows).
  - Verify the attestation’s own signature using the public key for `attestor_id` (resolved via RFC-0001). If signature verification fails, the attestation **MUST** be ignored.
  - For standalone attestation Packets (`content.type = "ATTESTATION"`):
    - `content.attestation.attestor_id` **MUST** equal the Packet `author_id`. If they differ, the attestation **MUST** be treated as invalid and ignored.
    - `content.attestation.target_packet` **MUST** reference the packet_id being attested; mismatches or malformed references **MUST** cause the attestation to be ignored.
  - For embedded attestations in `packet.attestations`:
    - `attestor_id` MAY be different from `author_id`, but the signature **MUST** verify against `attestor_id`.
  - Clients **MUST** treat embedded and standalone attestations that pass validation as equivalent inputs and **MUST** merge them by `(target_packet, attestation_id)` when computing quorums. Duplicate attestations (same `attestation_id` from the same attestor) **MUST NOT** be double-counted.
  - Attestations that fail any of the checks above (invalid signature, inconsistent `attestor_id`, mismatched `target_packet`) **MUST** be ignored for scoring and quorum purposes, but MAY be surfaced in advanced debugging views as “invalid.”

### 3.3 Attestation Retraction Packet

An attestor may retract one of its own attestations using `content.type = "ATTESTATION_RETRACTION"`:

```jsonc
{
  "content": {
    "type": "ATTESTATION_RETRACTION",
    "target_packet": "0x1e208f2c3a4b5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f7a1",
    "attestation_id": "0xatt2",
    "reason": "erroneous model output"
  },
  "signature": "0xAttestorSignature..."
}
```

Rules:
- Only the original attestor (matching the attested packet’s `attestor_id` / `author_id`) **MAY** issue a retraction; others are informational only and MUST NOT affect quorum weight.
- A retraction zeros the referenced attestation’s weight in quorum/reputation calculations while preserving auditability of the original claim.
- If multiple retractions exist from the attestor, the latest by relay‑observed `received_at` wins (tie‑break by lexicographically smallest `packet_id`).

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

Before applying any attestor policies or computing quorum weights, clients **MUST** drop attestations that fail the validation rules in §3.2 (signature and identity consistency checks). Only attestations that pass these checks are eligible to contribute to `R_total` or retroactive consensus.

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
- `A_support` **MUST** exclude attestations that have been retracted (via `ATTESTATION_RETRACTION` or `CORRECTION action: "retract"`).

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
The `attestations` array referenced in this section is the top-level `packet.attestations` field defined in RFC-0002 §3.1, not `content.attestations` or any other nested location. Implementations MUST NOT rely on attestations stored under `content` or other ad-hoc fields; such data is application-specific and out of scope for the attestation model.
