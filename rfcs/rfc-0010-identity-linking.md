# RFC-0010: Identity Linking Protocol

Status: Draft
Last updated: 2025-12-15

## 1. Purpose

This RFC defines a secure protocol for linking a Strata identity to a new device by scanning a visual code. The protocol enables users to:

- Transfer their identity to a new device without manually typing private keys
- Securely pair devices using a short-lived visual code + confirmation code
- Maintain end-to-end encryption of sensitive key material

The design is inspired by Apple Pay's "Scan Code with iPhone" feature, adapted for decentralized identity transfer.

## 2. Terminology

- **Source Device**: The device that currently holds the identity (Device A)
- **Target Device**: The new device receiving the identity (Device B)
- **Link Session**: A short-lived pairing session between two devices
- **Confirmation Code**: A 6-digit code derived from the shared secret for MITM protection
- **SIL**: Strata Identity Link (this protocol)

## 3. Security Requirements

The protocol MUST satisfy these security properties:

1. **No secrets in QR code**: The visual code MUST NOT contain private keys or long-term secrets
2. **Short-lived sessions**: Link sessions MUST expire within 60 seconds
3. **MITM protection**: A confirmation code MUST be verified to prevent man-in-the-middle attacks
4. **End-to-end encryption**: Private keys MUST be encrypted using ephemeral key exchange
5. **Forward secrecy**: Compromise of long-term keys MUST NOT compromise past link sessions
6. **User confirmation**: Both devices MUST display clear confirmation before completing the transfer

## 4. Protocol Overview

```
┌─────────────────┐                              ┌─────────────────┐
│   Source (A)    │                              │   Target (B)    │
│ (has identity)  │                              │ (new device)    │
└────────┬────────┘                              └────────┬────────┘
         │                                                │
         │ 1. INITIATE                                    │
         │    Generate session + ephemeral keypair        │
         │                                                │
         ▼                                                │
    ┌─────────┐                                           │
    │ QR Code │ ◄──────── 2. SCAN ────────────────────────┤
    │ (30sec) │                                           │
    └─────────┘                                           │
         │                                                │
         │                    3. CONNECT                  │
         │                       Generate ephemeral       │
         │                       keypair, connect WS      │
         │                                                ▼
         │           ◄─────── 4. LINK_REQUEST ───────────┤
         │                                                │
         │ 5. DERIVE                                      │
         │    Shared secret + confirmation code           │
         │                                                │
         ▼                                                │
    ┌─────────┐                                    ┌─────────┐
    │ 847291  │                                    │ [____]  │
    │ Confirm?│                                    │ Enter   │
    └─────────┘                                    └─────────┘
         │                                                │
         │           ◄─────── 6. LINK_CONFIRM ───────────┤
         │                    (with confirmation code)    │
         │                                                │
         │ 7. VERIFY + ENCRYPT                            │
         │                                                │
         │           ─────── 8. LINK_COMPLETE ──────────► │
         │                   (encrypted identity)         │
         │                                                │
         │                                    9. DECRYPT  │
         │                                       IMPORT   │
         ▼                                                ▼
    ┌─────────┐                                    ┌─────────┐
    │ Linked! │                                    │ Ready!  │
    └─────────┘                                    └─────────┘
```

## 5. Visual Code Format

### 5.1 QR Code Payload

The QR code MUST contain a JSON object with the following structure:

```json
{
  "type": "strata.identity.link",
  "v": 1,
  "sid": "<session_id>",
  "pk": "<base64_ephemeral_pubkey>",
  "relay": "<wss://relay_url>",
  "http": "<https://http_base_url>",
  "exp": <unix_timestamp>
}
```

Field definitions:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | YES | MUST be `"strata.identity.link"` |
| `v` | integer | YES | Protocol version. MUST be `1` for this RFC |
| `sid` | string | YES | Session ID (16 bytes, hex-encoded) |
| `pk` | string | YES | Ephemeral X25519 public key (32 bytes, base64url-encoded) |
| `relay` | string | YES | WebSocket relay URL for the link session |
| `http` | string | YES | HTTP base URL of the home server |
| `exp` | integer | YES | Expiry timestamp (Unix seconds) |

### 5.2 Session ID Generation

```
session_id = random_bytes(16).hex()
```

### 5.3 Ephemeral Key Generation

Source device MUST generate a fresh X25519 keypair for each link session:

```
(source_ephemeral_private, source_ephemeral_public) = X25519.generate_keypair()
```

### 5.4 QR Code Refresh

The QR code SHOULD refresh every 15-30 seconds with:
- New session ID
- New ephemeral keypair
- Updated expiry timestamp

Previous sessions SHOULD be invalidated upon refresh.

## 6. WebSocket Messages

Link sessions use the relay WebSocket connection with special message types.

All messages in this section use the RFC-0004 object framing with a top-level `type` field.

### 6.1 LINK_REQUEST (Source registration + Target → Relay → Source)

The source device MUST register the session with the relay before the target device connects:

```json
{
  "type": "LINK_REQUEST",
  "sid": "<session_id>",
  "pk": "<base64_source_ephemeral_pubkey>",
  "ts": <unix_timestamp>,
  "exp": <unix_timestamp>
}
```

The target device then connects after scanning the QR code:

```json
{
  "type": "LINK_REQUEST",
  "sid": "<session_id>",
  "pk": "<base64_target_ephemeral_pubkey>",
  "ts": <unix_timestamp>
}
```

### 6.2 LINK_CONFIRM (Target → Relay → Source)

Sent by target device after user enters the confirmation code:

```json
{
  "type": "LINK_CONFIRM",
  "sid": "<session_id>",
  "code": "<6_digit_confirmation_code>",
  "ts": <unix_timestamp>
}
```

### 6.3 LINK_COMPLETE (Source → Relay → Target)

Sent by source device after verifying the confirmation code:

```json
{
  "type": "LINK_COMPLETE",
  "sid": "<session_id>",
  "payload": "<base64_encrypted_identity>",
  "nonce": "<base64_nonce>",
  "ts": <unix_timestamp>
}
```

### 6.4 LINK_ERROR (Either → Relay → Other)

Sent when an error occurs:

```json
{
  "type": "LINK_ERROR",
  "sid": "<session_id>",
  "code": "<error_code>",
  "message": "<human_readable_message>"
}
```

Error codes:

| Code | Description |
|------|-------------|
| `session_expired` | The link session has expired |
| `session_not_found` | The session ID is not recognized |
| `invalid_code` | The confirmation code is incorrect |
| `cancelled` | User cancelled the link operation |
| `already_completed` | Session was already completed |

## 7. Cryptographic Operations

### 7.1 Shared Secret Derivation

Both devices derive a shared secret using X25519 key exchange:

```
shared_secret = X25519(my_ephemeral_private, their_ephemeral_public)
```

### 7.2 Key Derivation

Derive encryption key and confirmation code from shared secret:

```
# Using HKDF with SHA-256
derived = HKDF(
  ikm = shared_secret,
  salt = session_id_bytes,
  info = "strata.identity.link.v1",
  length = 48
)

encryption_key = derived[0:32]   # 32 bytes for AES-256-GCM
confirmation_code_bytes = derived[32:48]  # 16 bytes for code derivation
```

### 7.3 Confirmation Code Generation

Convert the first 4 bytes of `confirmation_code_bytes` to a 6-digit decimal:

```
code_int = int.from_bytes(confirmation_code_bytes[0:4], 'big') % 1000000
confirmation_code = str(code_int).zfill(6)  # e.g., "047291"
```

### 7.4 Identity Encryption

Encrypt the identity payload using AES-256-GCM:

```
nonce = random_bytes(12)
plaintext = json.dumps({
  "did": "<did:strata:...>",
  "public_key_hex": "<hex_public_key>",
  "private_key_hex": "<hex_private_key>"
})
ciphertext = AES-256-GCM.encrypt(encryption_key, nonce, plaintext, aad=session_id_bytes)
```

## 8. Session Management

### 8.1 Session States

```
PENDING    → CONNECTED → CONFIRMING → COMPLETED
    │            │            │
    └────────────┴────────────┴──────→ EXPIRED/CANCELLED
```

### 8.2 Timeouts

| Transition | Timeout |
|------------|---------|
| PENDING → CONNECTED | 30 seconds (QR expiry) |
| CONNECTED → CONFIRMING | 60 seconds |
| CONFIRMING → COMPLETED | 60 seconds |

### 8.3 Attempt Limits

- Maximum 3 incorrect confirmation code attempts per session
- After 3 failures, session MUST transition to CANCELLED

## 9. Relay Handling

Relays supporting identity linking MUST:

1. Accept `LINK_REQUEST`, `LINK_CONFIRM`, and `LINK_COMPLETE` message types
2. Route messages to the correct session based on `sid`
3. Enforce session expiry and clean up stale sessions
4. NOT persist or log encrypted identity payloads

Relays MAY implement rate limiting on link session creation.

## 10. User Interface Guidelines

### 10.1 Source Device (Linking Out)

1. Display prominent "Link New Device" button in security settings
2. Show animated QR code with visual refresh indicator
3. Display countdown timer showing session expiry
4. Show 6-digit confirmation code in large, readable font
5. Require explicit "Confirm" button press after code display
6. Show success/failure state clearly

### 10.2 Target Device (Linking In)

1. Provide "Scan to Link" option in login/import flow
2. Use native camera API or embedded scanner
3. Display parsed session info (relay, expiry) before proceeding
4. Show numeric keypad for confirmation code entry
5. Provide feedback on code verification status
6. Show imported identity info on success

### 10.3 Visual Design (Inspired by Apple Pay)

The QR code display SHOULD include:
- Concentric animated rings indicating "scannable" state
- Clear branding (Strata logo in center)
- High contrast for reliable scanning
- Accessibility considerations (audio cues, large text options)

## 11. Security Considerations

### 11.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| QR code photographed by attacker | Useless without source device to complete handshake |
| MITM intercepts QR scan | Confirmation code derived from E2E shared secret |
| Replay attack with old QR | Short session expiry (30 seconds) |
| Relay logs encrypted payload | E2E encryption; relay cannot decrypt |
| Brute force confirmation code | 3 attempt limit; 1-in-1-million odds per attempt |

### 11.2 Implementation Requirements

Implementations MUST:
- Use cryptographically secure random number generators
- Zeroize ephemeral private keys after use
- Clear confirmation codes from memory after verification
- Validate all message fields before processing
- Implement constant-time comparison for confirmation codes

### 11.3 Privacy Considerations

- Link sessions reveal that a user is performing device pairing
- The relay observes connection metadata but not key material
- Session IDs are random and do not correlate to DIDs until completion

## 12. Implementation Status

**Pending**:
- TypeScript SDK: `createLinkSession`, `scanLinkCode`, `completeLinkSession`
- StrataX: Link Device UI, Scan to Link UI
- Home Relay: Link session message routing

## 13. References

- RFC-0001: Identity and StrataID (key management)
- RFC-0004: Relay Transport Protocol (WebSocket messages)
- RFC-0009: Relay Connection URIs and QR Codes (QR format precedent)
- Apple Pay: Scan Code with iPhone (UX inspiration)
- Signal Protocol: X3DH (key exchange patterns)
