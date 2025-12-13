# RFC-0009: Relay Connection URIs and QR Codes

Status: Draft  
Last updated: 2025-12-13

## 1. Purpose

This RFC defines an **out-of-band** encoding for sharing relay connection details via:

- QR codes (scan-to-configure), and
- clickable links / deep links.

The goal is to let a user quickly configure a client to connect to a **home server** running:

- an HTTP API surface (e.g. `strata-home-relay` with embedded `/gate` and `/id`), and
- a relay WebSocket endpoint (RFC-0004).

This RFC does **not** change any wire protocol (RFC-0004). It only defines an interchange format.

## 2. Terminology

- **Home Server HTTP Base URL**: the origin used for HTTP endpoints (e.g. `https://home.example`).
- **Relay WS URL**: the WebSocket URL for RFC-0004 (e.g. `wss://home.example/strata`).
- **Primary relay**: the first-choice relay for publish/subscribe when multiple relays are configured.

## 3. Formats

Clients SHOULD support **both** formats:

1. **URI form** (recommended for QR + links)
2. **JSON form** (recommended for interoperability with non-URI channels)

Clients MAY support additional heuristics (e.g. scanning a plain `https://…` URL and treating it as the HTTP base URL).

### 3.1 URI form (recommended)

The canonical URI form is:

`strata://relay?v=1&http=<HTTP_BASE_URL>[&ws=<WS_URL>][&label=<LABEL>][&mode=<MODE>][&extra_ws=<WS_URL>...]`

Notes:

- The URI scheme is `strata`.
- The authority/host MUST be `relay`.
- Parameters are URL-encoded as per RFC 3986.
- `extra_ws` may be repeated to include additional relays in a pool.

#### 3.1.1 Parameters

Required:

- `v`: integer version. For this RFC, MUST be `1` when present. If omitted, clients MAY assume `1`.
- `http`: Home Server HTTP Base URL.
  - MUST be `http://…` or `https://…`
  - MUST NOT include a query string
  - SHOULD NOT include a path (clients MAY ignore a path if present)

Optional:

- `ws`: Primary Relay WS URL.
  - MUST be `ws://…` or `wss://…`
  - If omitted, clients SHOULD derive it from `http` by:
    - converting `https://` → `wss://` or `http://` → `ws://`
    - appending `/strata`
- `label`: human-friendly label for the relay/server (for UI only).
  - Clients SHOULD cap length (e.g. 64 chars) and strip control characters.
- `mode`: configuration hint: `auto` or `manual`.
  - `auto`: client MAY continue to use bootstrap/discovery mechanisms.
  - `manual`: client SHOULD treat this as user-pinned configuration.
- `extra_ws`: additional relay WS URLs to add to the relay pool (publish failover, read redundancy).
  - May be repeated: `&extra_ws=wss://...&extra_ws=wss://...`

#### 3.1.2 Examples

Minimal (HTTP only):

`strata://relay?v=1&http=https%3A%2F%2Fhome.example`

Explicit WS + label:

`strata://relay?v=1&http=https%3A%2F%2Fhome.example&ws=wss%3A%2F%2Fhome.example%2Fstrata&label=Home`

Pool (primary + extra relays):

`strata://relay?v=1&http=https%3A%2F%2Fhome.example&ws=wss%3A%2F%2Fhome.example%2Fstrata&extra_ws=wss%3A%2F%2Fbackup.example%2Fstrata`

### 3.2 JSON form (recommended fallback)

The JSON form is a UTF-8 JSON object encoded directly into the QR code (or shared via text):

```json
{
  "type": "strata.relay",
  "v": 1,
  "http": "https://home.example",
  "ws": "wss://home.example/strata",
  "label": "Home",
  "mode": "auto",
  "extra_ws": ["wss://backup.example/strata"]
}
```

Rules:

- `type` MUST be `strata.relay`
- `v` MUST be `1`
- `http` is REQUIRED and follows the URI rules
- `ws`, `label`, `mode`, and `extra_ws` are OPTIONAL and follow the URI semantics

## 4. Security considerations

Relay QR payloads are **untrusted input**.

Clients SHOULD:

- Display the target host(s) and require explicit user confirmation before applying changes.
- Prefer `https`/`wss` by default and warn when `http`/`ws` is used (dev-mode exception).
- Treat `label` as untrusted (no rich text; no control chars).
- Avoid automatically enabling unknown relays for publishing without user intent.

## 5. IANA considerations

None.

