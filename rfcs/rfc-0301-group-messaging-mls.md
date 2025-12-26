<!-- rfcs/rfc-0301-group-messaging-mls.md -->

# RFC-0301: Group Messaging (MLS) (Optional)

- **Status:** Draft
- **Scope:** Optional E2EE group messaging profile
- **Author(s):** Strata Core Team
- **Created:** 2025-12-25

## 1. Summary

This RFC defines an optional group messaging profile built on MLS (Messaging Layer Security, RFC 9420). It extends Strata's existing MESSAGE transport (RFC-0004) without introducing new relay message types. Group state and encryption are managed client-side.

This RFC is opt-in and experimental until accepted. Clients MUST NOT claim interoperable group messaging unless they fully implement this profile.

## 2. Goals

- Provide an interoperable E2EE group messaging model.
- Reuse existing relay transport and Packet schema (RFC-0002/0004).
- Support multi-device membership (each device is a member).
- Keep group membership changes auditable by members via MLS.

## 3. Non-goals

- Metadata anonymity (relay and network observers still see MESSAGE traffic).
- Relay-side fanout or special routing behavior.
- Guaranteed forward secrecy against compromised clients (handled by MLS but not absolute).

## 4. Dependencies

- RFC-0000: Terminology and conventions
- RFC-0001: StrataID and key material
- RFC-0002: Packets and signatures
- RFC-0004: Relay transport protocol
- RFC-0200: Content types
- IS-0001: Direct Messages (E2EE) v1 (baseline MESSAGE handling)
- RFC 9420: MLS (Messaging Layer Security)

## 5. Terminology

- **Group ID:** Stable identifier for a group (32-byte random value).
- **Epoch:** MLS group state version.
- **Key Package:** MLS KeyPackage used to add a member.
- **Welcome:** MLS Welcome message used to initialize a new member.
- **MLS Message:** Any MLS ciphertext (application, commit, proposal).

## 6. Threat Model

Assumes:
- Relays can observe MESSAGE packet metadata and timing.
- Network observers can correlate IP and timing.
- Some group members may be compromised.

Out of scope:
- Compromised client devices or key theft.
- Global adversaries with full visibility of all relay traffic.

## 7. Design Overview

Group messaging uses MLS for key management and encryption. The MLS ciphertext is carried in Strata MESSAGE packets. Relays treat MESSAGE packets as opaque and forward according to RFC-0004 policy.

### 7.1 Group ID

Group IDs MUST be 32 bytes of cryptographically random data, encoded as `0x` + lowercase hex.

Example:

```text
0x5f4dcc3b5aa765d61d8327deb882cf99b73f167239b1c2d1f7a6d84d2b6c4a11
```

### 7.2 Key package publication

Group-capable clients MUST publish MLS key packages so others can add them to groups.

Packet requirements:
- `author_id` MUST be the DID of the publishing device's owner.
- `content.type` MUST be `"strata.messaging.key_package.v1"`.

Content shape:

```jsonc
{
  "type": "strata.messaging.key_package.v1",
  "key_package_version": 1,
  "device_key_id": "did:strata:alice#keys-2",
  "mls_cipher_suite": "MLS_128_DHKEMX25519_AES128GCM_SHA256_Ed25519",
  "key_package_b64": "<base64url(MLS KeyPackage)>",
  "created_at": 1715421200,
  "expires_at": 1718013200
}
```

Rules:
- `key_package_version` MUST be `1`.
- `device_key_id` MUST be a DID verification-method identifier belonging to `author_id`.
- `key_package_b64` MUST be a valid MLS KeyPackage for the declared cipher suite.
- `expires_at` SHOULD be set (recommended: `created_at + 90 days`).

### 7.3 Welcome delivery

To add a new member, clients MUST deliver the MLS Welcome message to the new member using a Strata MESSAGE packet.

The MESSAGE packet:
- MUST use `content.type = "MESSAGE"`.
- MUST include an encryption scheme of `"mls-welcome-v1"`.
- MUST target the new member device (similar to IS-0001 recipient delivery).

Content shape:

```jsonc
{
  "type": "MESSAGE",
  "encryption": {
    "scheme": "mls-welcome-v1",
    "group_id": "0x...",
    "epoch": 1,
    "welcome_b64": "<base64url(MLS Welcome)>"
  }
}
```

Rules:
- `group_id` MUST be present.
- `epoch` MUST be present.
- `welcome_b64` MUST be a valid MLS Welcome for the declared group and epoch.
- Clients MUST ignore Welcomes that do not validate against local MLS policy.

### 7.4 Group messages

Group messages are carried as MESSAGE packets containing MLS ciphertext:

```jsonc
{
  "type": "MESSAGE",
  "encryption": {
    "scheme": "mls-message-v1",
    "group_id": "0x...",
    "epoch": 12,
    "ciphertext_b64": "<base64url(MLS ciphertext)>"
  }
}
```

Rules:
- `group_id` MUST be present.
- `epoch` MUST be present.
- `ciphertext_b64` MUST be a valid MLS ciphertext for the group and epoch.

Clients MUST reject messages if the MLS ciphertext fails to validate for the local group state.

### 7.5 Multi-device membership

Each device SHOULD be treated as a distinct MLS member.

Clients SHOULD:
- Add each device using its published key package.
- Remove compromised devices via MLS remove proposals.
- Re-key after membership changes (MLS commit).

### 7.6 Membership attestations

Membership changes SHOULD be auditable by group participants.

Clients SHOULD publish `GROUP_MEMBERSHIP` packets (RFC-0200) when:
- A member is added (`action = "ADD"`).
- A member is removed (`action = "REMOVE"`).

These packets are not required for MLS security, but they help participants audit who was added or removed. Because membership packets reveal group relationships, clients MAY omit them for high-privacy groups and MUST communicate that limitation in the UI.

### 7.7 Retention and deletion

- MESSAGE packets MAY include `expires_at` to request time-bound retention.
- Clients MUST enforce `expires_at` locally and SHOULD warn users that relay-side deletion is best effort.
- Relays SHOULD honor `expires_at` when enforcing retention policies.

## 8. UX Requirements

Clients exposing group messaging:

- MUST explain that relay and network metadata remains visible.
- SHOULD show a clear warning when a member is added or removed.
- SHOULD expose "leave group" and "report abuse" actions.

## 9. Compatibility

This RFC does not require new relay message types. All group messages are standard Packets with `content.type = "MESSAGE"`.

Relays that do not recognize MLS-specific envelopes MUST still store and forward MESSAGE packets as opaque data.

## 10. Security Considerations

- MLS provides forward secrecy and post-compromise security when correctly implemented, but only for members who receive updates.
- Long-lived devices should rotate key packages regularly.
- Clients SHOULD treat MLS epoch rollback as a security event.

## 11. Open Questions

- Standardized mapping for MLS credentials to StrataIDs.
- Optional group metadata packets (name, avatar) with privacy-preserving defaults.
- Relay-side retention policies for large group traffic.
