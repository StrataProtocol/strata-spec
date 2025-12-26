<!-- rfcs/rfc-0302-zk-disclosure-and-private-reputation.md -->

# RFC-0302: ZK Disclosure and Private Reputation (Optional)

- **Status:** Draft
- **Scope:** Optional selective disclosure profile
- **Author(s):** Strata Core Team
- **Created:** 2026-01-15

## 1. Summary

This RFC defines an optional, experimental profile for zero-knowledge (ZK) disclosure in the Strata Protocol. The goal is to allow clients and services to prove limited facts (for example, "verification level >= phone" or "at least 3 independent attestations") without revealing full identity, full attestation lists, or sensitive underlying data.

This RFC does not add new relay message types and does not change RFC-0004 transport semantics. It specifies an optional disclosure envelope carried in `content.app_metadata` so that relays can treat proofs as opaque data.

## 2. Goals

- Provide a privacy-preserving disclosure mechanism for selective facts.
- Support threshold-based reputation proofs without exposing full evidence sets.
- Remain optional and backwards compatible with existing Strata clients/relays.
- Keep the user experience clear: proofs are signals, not absolute truth.

## 3. Non-goals

- Global anonymity or metadata privacy (see RFC-0300).
- On-chain proof publication or blockchain dependencies.
- Mandatory adoption by relays or clients.

## 4. Dependencies

- RFC-0000: Canonical JSON
- RFC-0002: Packet structure
- RFC-0003: Attestations
- RFC-0005: Trust, reputation, and Reality Tuner

## 5. Terminology

- **Statement:** The claim being proven (for example, "verification_level >= phone").
- **Proof System:** The cryptographic scheme used to generate/verify the proof.
- **Public Inputs:** Data needed to verify a proof without revealing private inputs.

## 6. Disclosure Envelope (Optional)

Clients MAY attach a ZK disclosure envelope to any Packet by placing a `zk_disclosure` object inside `content.app_metadata`.

Example:

```jsonc
{
  "type": "ATTESTATION",
  "app_metadata": {
    "zk_disclosure": {
      "statement": "strata.zk.identity.verification_level.v1",
      "proof_system": "groth16-bn254",
      "proof_b64": "<base64url(proof)>",
      "public_inputs": {
        "min_level": "phone",
        "subject_hash": "0x..."
      },
      "created_at": 1715421200,
      "expires_at": 1718013200
    }
  }
}
```

Rules:
- `statement` MUST be a namespaced, versioned identifier (for example, `strata.zk.identity.verification_level.v1`).
- `proof_system` MUST be a string identifying the scheme and curve (if applicable).
- `proof_b64` MUST be base64url-encoded.
- `public_inputs` MUST be an object containing only JSON-serializable values and MUST be canonicalized per RFC-0000 when generating and verifying proofs.
- `created_at` MUST be a Unix epoch seconds timestamp.
- `expires_at` SHOULD be set for time-bounded proofs.

Clients that do not understand the statement or proof system MUST ignore the proof and treat the Packet as if no disclosure were present.

## 7. Recommended Statements (Non-Normative)

Implementations should use explicit, versioned statement identifiers. Suggested initial registry:

| Statement ID | Meaning | Host Packet |
|-------------|---------|-------------|
| `strata.zk.identity.verification_level.v1` | Subject verification level >= threshold | ATTESTATION |
| `strata.zk.attestation.threshold.v1` | Content has >= N independent attestations | CONSENSUS_METRIC |
| `strata.zk.relay.reputation.v1` | Relay reputation score >= threshold | RELAY_DESCRIPTOR |
| `strata.zk.membership.set.v1` | Subject is a member of a declared set | ATTESTATION |

Statements MUST clearly define their `public_inputs` schema and verification requirements in a statement-specific spec or manifest.

## 8. Verification Behavior

- Verifiers MUST validate the proof against the statement definition and public inputs before presenting the disclosure as meaningful.
- Clients SHOULD surface disclosures as attributable signals ("Proof shows...") and avoid absolute claims.
- If verification fails, clients MUST treat the disclosure as invalid and SHOULD show an "unverified proof" warning in advanced views.

## 9. UX Requirements

- Disclosures SHOULD be optional and hidden in non-advanced UI by default.
- User-facing copy MUST avoid implying anonymity or certainty.
- When a disclosure is used to filter or rank content, clients MUST explain that this is a user-configurable signal.

## 10. Security Considerations

- Proof systems with trusted setup introduce operational risk. Implementations SHOULD document their setup ceremonies.
- A valid proof only asserts the statement; it does not guarantee truth outside that scope.
- Replay risk: proofs SHOULD be time-bounded via `expires_at` and bound to a subject hash or context.

## 11. Open Questions

- Standardized statement registry governance.
- Proof format negotiation between clients/attestors.
- How to express revocation or updates for long-lived proofs.
