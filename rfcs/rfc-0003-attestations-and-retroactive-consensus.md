<!-- rfcs/rfc-0003-attestations-and-retroactive-consensus.md -->

# RFC-0003: Attestations and Retroactive Consensus

- **Status:** Draft
- **Author(s):** Strata Core Team
- **Created:** 2025-11-25
- **Updated:** 2025-11-28
- **Scope:** Normative protocol (attestations, retroactive consensus wire semantics)

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
    "target_packet": "0x1e208f2c3a4b5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f7a1",
    "domain": "PROVENANCE",         // PROVENANCE | CONTENT | SPAM_ABUSE | OTHER
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
- `target_packet`: `packet_id` of the Packet being attested. **REQUIRED** for all attestations.
- `domain` (optional): what category of claim this is about. MUST be one of `PROVENANCE`, `CONTENT`, `SPAM_ABUSE`, or `OTHER`. If omitted, clients **MUST** infer it from `subject` using §4.4 to preserve backwards compatibility. Unknown enum values **MUST** be safely ignored per RFC‑0000 §4.
- `subject`: claim subject (e.g., ORIGIN_LIKELY_HUMAN).
- `confidence`: numeric confidence in [0,1].
- `method`: free‑form description or version string.
- `issued_at`: timestamp.
- `metadata`: optional structured data.
- `signature`: attestor's signature over the attestation payload.

For all attestations (embedded or standalone), the signed payload **MUST** include a `target_packet` field whose value is the `packet_id` of the Packet being attested. Attestations without `target_packet` in the signed payload **MUST** be ignored.

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
      "domain": "PROVENANCE",
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
- Before an attestation contributes to any Trust Engine / Reality Tuner, quorum, or reputation logic, clients **MUST**:
  - Validate the containing Packet per RFC-0002 (packet signature, `author_id`, and time windows).
  - Verify the attestation’s own signature using the public key for `attestor_id` (resolved via RFC-0001). If signature verification fails, the attestation **MUST** be ignored.
  - For standalone attestation Packets (`content.type = "ATTESTATION"`):
    - `content.attestation.attestor_id` **MUST** equal the Packet `author_id`. If they differ, the attestation **MUST** be treated as invalid and ignored.
    - `content.attestation.target_packet` **MUST** reference the packet_id being attested; mismatches or malformed references **MUST** cause the attestation to be ignored.
  - For embedded attestations in `packet.attestations`:
    - `target_packet` **MUST** equal the host Packet's `packet_id`. Clients **MUST** ignore embedded attestations whose `target_packet` does not match the host `packet_id`.
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
- A retraction **MUST** zero the referenced attestation’s weight in quorum/reputation calculations while preserving auditability of the original claim.
- If multiple retractions exist from the attestor, the latest by relay‑observed `received_at` wins (tie‑break by lexicographically smallest `packet_id`).

### 3.4 Attestation Signature Scope

The `signature` field in an Attestation object **MUST** be computed over the canonical JSON representation of the Attestation object with the `signature` field omitted, using the canonicalization rules from RFC-0000 §4.2. The signed payload **MUST** include at minimum: `attestation_id`, `attestor_id`, `target_packet`, `subject`, `confidence`, and `issued_at`.

This binding prevents claim-rebinding attacks where an attestation is copied from one Packet to another. Because `target_packet` is included in the signed payload, any attempt to embed the attestation in a different Packet will cause signature verification to fail.

### 3.5 Append-Only Attestation History

Attestations, corrections, and retractions are append‑only:
- Existing attestation objects are **never** edited in place.
- Corrections and `ATTESTATION_RETRACTION` Packets are expressed as new Packets referencing the original attestation.
- Clients **MUST** keep the historical record while applying the latest valid retraction/correction from the original attestor to compute effective weight.

## 4. Claim Types

Attestations are explicitly two-dimensional: `domain` (what kind of thing is being judged) and `subject` (the specific claim). Clients MAY introduce additional subjects over time; unknown values MUST be ignored per RFC‑0000 §4.

### 4.1 Domain: PROVENANCE
- `ORIGIN_LIKELY_HUMAN`  
  Analysis suggests the media was likely created by a human (non‑AI) workflow.
- `ORIGIN_LIKELY_SYNTH`  
  Analysis suggests the media was likely created by an AI model.
- `MANIPULATED`  
  Pixels / audio have been materially altered post‑capture in a way that changes the apparent meaning (e.g., face swap, splice). Distinct from benign cropping/brightness tweaks.
- `UNALTERED_HARDWARE_CAPTURE`  
  For content with a full hardware chain: no edits detected beyond benign transformations, and the chain validates.

### 4.2 Domain: CONTENT
- `FACTUAL_INACCURACY`  
  One or more core factual claims in the packet are verifiably false, given the attestor’s evidence.
- `OUT_OF_CONTEXT`  
  The media is real but the claimed time / place / event is wrong (e.g., old protest clip presented as new).
- `CAPTION_MISLEADING`  
  Caption/headline/overlay materially misdescribes what’s in the media, even if the raw media is real and in context.
- `MISATTRIBUTED_SOURCE`  
  Wrong identity or organization credited or blamed for the content or event.
- `FABRICATED_EVENT`  
  The narrative describes an event that the attestor asserts did not happen at all (e.g., “explosion in X city today” where no such event occurred).

### 4.3 Domain: SPAM_ABUSE
- `SPAM`  
  Low‑value, repetitive, or automated promotional/garbage content.
- `ABUSIVE`  
  Targeted harassment, slurs, threats, etc.
- `SCAM`  
  Financial or other fraud attempts.

### 4.4 Domain inference (backwards compatibility)
If `domain` is absent, clients **MUST** infer it using:

```text
PROVENANCE ← {ORIGIN_LIKELY_HUMAN, ORIGIN_LIKELY_SYNTH, MANIPULATED, UNALTERED_HARDWARE_CAPTURE}
CONTENT    ← {FACTUAL_INACCURACY, OUT_OF_CONTEXT, CAPTION_MISLEADING, MISATTRIBUTED_SOURCE, FABRICATED_EVENT}
SPAM_ABUSE ← {SPAM, ABUSIVE, SCAM}
```

Implementations SHOULD emit `domain` going forward; legacy attestations without it remain valid through this mapping. Each claim type may have its own thresholds and handling in clients.

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
- `A_support(C)` = set of attestations about `P` whose `subject = C` (e.g., `MANIPULATED`) that have **not** been retracted (via `ATTESTATION_RETRACTION` or `CORRECTION action: "retract"`),
- `R_total(i)` = reputation score of attestor `i` (see RFC‑0005).

`P` reaches quorum **for claim `C`** when all are true:

1. `|A_support(C)| ≥ N_min(C)` (distinct attestors),
2. `Σ_{a in A_support(C)} R_total(attestor(a)) ≥ W_min(C)` (weighted support),
3. Attestors in `A_support(C)` span at least `C_min(C)` distinct trust clusters (not all from one tight community),
4. The oldest attestation in `A_support(C)` (by relay‑observed `received_at`) is at least `T_min(C)` seconds old (to resist flash‑mobs).

Thresholds (`N_min`, `W_min`, `C_min`, `T_min`) are configured **per claim type** and **per Reality Switch mode** (Strict / Standard / Wild). Defaults live in RFC‑0005 as reference profiles, not protocol requirements.

Conflicting quorums are possible (e.g., `MANIPULATED` and `UNALTERED_HARDWARE_CAPTURE` each hitting their own thresholds under different attestor sets). Reference handling is described in §7.4.

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

Clients **MUST NOT** be required by the protocol to delete or permanently block P. Retroactive consensus is advisory to the Trust Engine / Reality Tuner.

### 7.4 Reference Conflict Presentation (Non-normative)

Reference clients SHOULD:
- Display all non‑retracted attestations from attestors the user trusts on a given Packet.
- When multiple trusted attestations about the same subject disagree (e.g., `MANIPULATED` vs `UNALTERED_HARDWARE_CAPTURE`), mark the Packet as **contested** and surface the disagreement (e.g., “3 labs say manipulated, 1 lab says unaltered”).
- In Strict/Standard modes, prefer the more cautious label when conflicting claims each meet quorum (e.g., treat `MANIPULATED` as dominant over `UNALTERED`), while still exposing the full attestation set to the user.

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
- Unknown `domain`, `subject`, or `attestor_type` values MUST be safely ignored by clients, treated as unrecognized but not necessarily untrusted.
- Attestations without `domain` MUST continue to function via the inference table in §4.4.

### 10. Reference Implementation Notes

Reference client **SHOULD**:
- Cache attestations per `target_packet`,
- Allow user to inspect all attestations for a given Packet,
- Expose settings to choose which attestors to trust.
The `attestations` array referenced in this section is the top-level `packet.attestations` field defined in RFC-0002 §3.1, not `content.attestations` or any other nested location. Implementations MUST NOT rely on attestations stored under `content` or other ad-hoc fields; such data is application-specific and out of scope for the attestation model.
