<!-- rfcs/rfc-0008-attestor-capability-manifest-and-evidence-metadata.md -->

# RFC-0008: Attestor Capability Manifest and Evidence Metadata

- **Status:** Draft
- **Author(s):** Strata Core Team
- **Created:** 2025-12-12
- **Updated:** 2025-12-12
- **Scope:** Reference interoperability conventions (attestor transparency, evidence metadata); does **not** change core Packet/Attestation wire formats.

## 1. Summary

This RFC defines **optional** conventions to make attestors more interoperable and auditable:

1. An HTTP-hosted **Attestor Manifest** (`/.well-known/strata/attestor.json`) describing an attestor’s identity, capabilities, and evidence endpoints.
2. A recommended **`method` naming format** for Attestation objects (RFC‑0003).
3. A recommended **`metadata` schema** for Attestation objects to carry:
   - toolchain outputs (detectors/verifiers),
   - provenance checks,
   - evidence references (report hashes/URIs),
   while keeping Packets small (RFC‑0002 §3.5).

These conventions are not required for protocol compliance. Unknown fields MUST be ignored by clients per RFC‑0000 §4.

## 2. Motivation

RFC‑0003 standardizes **what** an Attestation is (signed claims about Packets), but intentionally leaves:

- `method` semantics as free-form,
- `metadata` structure as free-form,
- evidence hosting / transparency details out of scope.

For real attestor ecosystems (labs, NGOs, model providers, enterprise deployments), we want:

- A simple way for humans and systems to discover what an attestor does,
- Consistent, machine-readable evidence summaries,
- Optional deeper evidence retrieval for audits and disputes,
- Minimal client coupling: clients can remain compatible by ignoring unknown metadata.

## 3. Definitions

- **Attestor**: An entity publishing RFC‑0003 attestations.
- **Tool**: A detector/verifier used by an attestor (ML model, heuristic, watermark detector, C2PA verifier, etc.).
- **Analysis Report**: A structured artifact describing how an attestation was produced (often larger than Packet limits).
- **Evidence Reference**: A content-addressed reference (e.g., `0x…` hash and optional URI) to an Analysis Report or other artifact.

## 4. Attestor Manifest (`/.well-known/strata/attestor.json`)

### 4.1 Location

Attestors MAY publish a JSON manifest at:

```text
https://<attestor-host>/.well-known/strata/attestor.json
```

If present:
- It MUST be served over HTTPS.
- It MUST be valid JSON.
- It SHOULD be served with `Content-Type: application/json; charset=utf-8`.

### 4.2 Manifest Schema (v1)

If an attestor publishes a manifest, it SHOULD conform to the following structure.

```jsonc
{
  "schema": "strata:attestor-manifest:v1",
  "attestor_id": "did:strata:z6Mkn...",
  "attestor_type": "LAB",

  "display_name": "Example Forensics Lab",
  "description": "Automated + human-in-the-loop media forensics for Strata.",
  "website": "https://lab.example",

  "contact": {
    "email": "security@lab.example",
    "support": "https://lab.example/support"
  },

  "jurisdiction": {
    "country": "US",
    "region": "CA",
    "legal_entity": "Example Lab, Inc."
  },

  "capabilities": {
    "domains": ["PROVENANCE", "CONTENT", "SPAM_ABUSE"],
    "subjects": ["ORIGIN_LIKELY_SYNTH", "MANIPULATED", "OUT_OF_CONTEXT"],
    "media_types": ["image/jpeg", "image/png", "video/mp4", "audio/mpeg"],
    "modes": ["auto", "human", "hybrid"]
  },

  "evidence": {
    "reports_public": false,
    "report_retention_days": 30,
    "redaction_policy": "May omit raw media; see report.redactions."
  },

  "endpoints": {
    "api_base": "https://attestor.example/api/v1",
    "attestation_lookup": "/attestations/{attestation_id}",
    "report_lookup": "/reports/{report_id}"
  },

  "signed_at": 1760000000,
  "signature": "0x..."
}
```

Field notes:
- `schema` MUST be `"strata:attestor-manifest:v1"` for this version.
- `attestor_id` MUST be the attestor StrataID (RFC‑0001).
- `attestor_type` MUST be one of RFC‑0003 `attestor_type` values.
- `capabilities.subjects` SHOULD list RFC‑0003 subjects the attestor may emit.
- `endpoints.*` are OPTIONAL and purely informational unless a client/operator chooses to use them.

### 4.3 Manifest Signature (RECOMMENDED)

If a manifest includes a `signature` field, it SHOULD be computed as:
- Ed25519 signature over the canonical JSON of the manifest with `signature` omitted,
- Using the attestor’s active signing key (resolved by `attestor_id` per RFC‑0001).

Clients/operators MAY verify the manifest signature for stronger binding between the HTTP host and the claimed `attestor_id`.

If a manifest contains a `signature` but verification fails, clients/operators SHOULD treat the manifest as untrusted and ignore it.

## 5. Attestation `method` Naming Convention

RFC‑0003 defines `method` as free-form. For interoperability, this RFC RECOMMENDS:

```text
<vendor>/<toolchain>[:<component> ]@<semver>
```

Examples:
- `strata-labs/auto-attestor@1.0.0`
- `example-lab/forensics-suite:ela@2.3.1`
- `model-provider-x/watermark-detector@synthid-0.9.2`

Notes:
- `method` SHOULD be stable for a given analysis pipeline version to support auditing.
- Attestors SHOULD NOT embed secrets or personally identifying information in `method`.

## 6. Attestation `metadata` Schema (Recommended)

RFC‑0003 allows `metadata` to be an arbitrary object. This section defines a recommended shape for **tool-based attestations** (auto/hybrid).

### 6.1 Size Constraints

Because attestations are embedded in Packets (RFC‑0002), attestors:
- SHOULD keep `metadata` small (RECOMMENDED: ≤ 8 KiB).
- SHOULD store large details in an external Analysis Report and include only content-addressed references.

### 6.2 `metadata.schema`

If an attestor follows this schema, it SHOULD set:

```json
{ "schema": "strata:attestation-metadata:v1" }
```

### 6.3 Recommended Fields (v1)

```jsonc
{
  "schema": "strata:attestation-metadata:v1",

  "analysis": {
    "analysis_id": "0x...",              // stable id for the run (implementation-defined)
    "pipeline": "strata-labs/auto-attestor@1.0.0",
    "performed_at": 1760000000
  },

  "input": {
    "target_packet": "0x...",
    "media": [
      {
        "media_hash": "0xaf1349b9f5f9a1a6a0404dea36dcc9499bcb25c9adc112b7cc9a93cae41f3262",
        "media_type": "image/jpeg",
        "size_bytes": 238472,
        "hash_verified": true
      }
    ],
    "text_present": true
  },

  "provenance": {
    "origin_type": "UNKNOWN",
    "genesis_media_hash": null,
    "checks": [
      { "check": "c2pa_manifest", "status": "not_present" },
      { "check": "edit_history_chain", "status": "not_applicable" }
    ]
  },

  "tools": [
    {
      "tool": "builtin-ela",
      "version": "1.0.0",
      "capability": "manipulation",
      "label": "manipulated",
      "score": 0.91,
      "weight": 0.5
    }
  ],

  "aggregate": {
    "subject": "MANIPULATED",
    "score": 0.91,
    "threshold": 0.70
  },

  "evidence": {
    "report_hash": "0x...",              // hash of the analysis report bytes
    "report_uri": "ipfs://...",          // optional fetch URI
    "report_mime": "application/json",
    "redactions": ["raw_media_not_included"]
  }
}
```

Notes:
- `tools[*].score` SHOULD be in [0,1] and SHOULD correspond to the tool’s confidence for the returned `label`.
- `aggregate.subject` SHOULD match the RFC‑0003 `subject` emitted in the attestation itself (the attestation remains authoritative).
- `report_uri` is OPTIONAL; if present, clients/operators MAY fetch and validate bytes against `report_hash`.

## 7. Analysis Report (Optional External Artifact)

Attestors MAY produce an Analysis Report to support audits, disputes, and reproducibility.

### 7.1 Report Content (Recommended)

If produced, a report SHOULD include at least:
- `report_schema` + version,
- `attestation_id`,
- `attestor_id`,
- `target_packet`,
- input summary (media refs, sizes),
- tool outputs (including errors/timeouts),
- aggregation and decision details,
- redactions and retention notes.

### 7.2 Report Hashing

If an attestor publishes `metadata.evidence.report_hash`, it SHOULD be computed as:

```text
report_hash = "0x" + hex(blake3-256(report_bytes))
```

If `report_uri` is present and fetchable, the retrieved bytes MUST match `report_hash` for the report to be considered valid.

## 8. Security & Privacy Considerations

- **No PII in attestations:** Attestors SHOULD NOT include personally identifying information in `metadata` or reports unless strictly necessary and explicitly consented.
- **Untrusted inputs:** Tools often process attacker-controlled bytes; attestors SHOULD sandbox tools, enforce timeouts, and validate media types/sizes.
- **Evidence hosting:** Publishing full reports may expose sensitive details (faces, locations, copyrighted content). Attestors SHOULD support redactions and configurable retention.
- **Non-oracle framing:** Clients SHOULD present attestations as attributable signals (“Lab X says…”) not absolute truth.

## 9. Backwards Compatibility

- All fields defined in this RFC are optional.
- Clients MUST ignore unknown fields and unknown schemas per RFC‑0000 §4.
- Attestors MUST continue to produce RFC‑0003-compliant attestations even if they omit these metadata conventions.
