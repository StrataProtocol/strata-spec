<!-- rfcs/rfc-0200-social-content-types-v1.md -->

# RFC-0200: Social Content Types v1

- **Status:** Draft  
- **Author(s):** Strata Core Team  
- **Created:** 2025-11-27  
- **Updated:** 2025-11-28  
- **Scope:** Normative protocol (shared social content schemas for `content.type`)

## 1. Summary

Standardizes social content types and shapes for Strata apps (STRATAX, STRATAZ, STRATAF, CTRL+F5):
- Extends `content.type = "POST"` semantics,
- Introduces `PROFILE`, `SOCIAL_EDGE`, `SOCIAL_EDGE_REVOCATION`, `REACTION`, `GROUP`, `GROUP_MEMBERSHIP`, `LIST`, `LIST_MEMBERSHIP`,
- Encourages reuse across apps instead of bespoke schemas.

Designed to interoperate with Packets/provenance (RFC‑0002), identity/trust edges (RFC‑0001), attestations (RFC‑0003), Trust Engine (RFC‑0005), and relay transport (RFC‑0004).

## 2. Motivation

Shared schemas avoid fragmentation of posts, profiles, reactions, and social graphs across apps. Goals:
- One POST shape for tweets/status updates/short videos/news articles,
- One PROFILE shape all apps can render,
- Clear separation of trust graph (TRUST_EDGE) vs. social graph (SOCIAL_EDGE).

## 3. Conventions

- Uses terminology/canonicalization from RFC‑0000 and Packet structure from RFC‑0002.
- Unknown fields **MUST** be ignored.
- Unknown `content.type` values **MUST** be safely ignored (fallback UI).
- Timestamps are Unix epoch seconds.

## 4. Common building blocks

### 4.1 Media references

Use `media` array from RFC‑0002 §3.2; include `provenance_header` when non‑empty. Media entries MAY extend with:

```jsonc
{
  "media_hash": "0xaf1349b9f5f9a1a6a0404dea36dcc9499bcb25c9adc112b7cc9a93cae41f3262",
  "media_type": "video/mp4",
  "size_bytes": 123456,
  "media_uris": ["ipfs://Qm...", "https://cdn.example/media/Qm..."],
  "duration_seconds": 12.3,
  "width": 1080,
  "height": 1920,
  "variants": [
    {
      "label": "thumb",
      "media_hash": "0x0b68f63dfb0a3b5e4b8c6e2a7e0d7d2c0cc1b5c7a8d9e6f1a2b3c4d5e6f70819",
      "media_type": "image/webp",
      "width": 320,
      "height": 568
    },
    {
      "label": "720p",
      "media_hash": "0x8f2a4c6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f708192a",
      "media_type": "video/mp4",
      "width": 720,
      "height": 1280,
      "bitrate_kbps": 1800
    }
  ],
  "alt_text": "short description for accessibility"
}
```

Clients **MUST** ignore unknown fields in media objects.

### 4.2 App metadata

All social content types MAY include an `app_metadata` object:

```jsonc
"app_metadata": {
  "app": "STRATAX",
  "version": "1.0.0",
  "extras": {
    "pin_to_profile": true
  }
}
```

`app` is a short string (e.g., `STRATAX`, `STRATAZ`, `STRATAF`, `CTRL+F5`). `extras` is free‑form; unknown keys **MUST** be ignored.

## 5. POST – social posts, videos, and news

### 5.1 Type and overview

`content.type = "POST"`; fields standardized by this RFC:

```jsonc
{
  "type": "POST",
  "post_kind": "ORIGINAL",
  "text": "Hello Strata",
  "media": [ ... ],
  "thread_ref": null,
  "quote_ref": null,
  "repost_of": null,

  "mentions": ["did:strata:alice"],
  "hashtags": ["strata", "launch"],
  "language": "en",
  "audience": "PUBLIC",

  "link_previews": [ ... ],
  "article": { ... },
  "short_video": { ... },

  "group_ref": null,

  "app_metadata": { ... }
}
```

All fields OPTIONAL unless noted.

### 5.2 post_kind

Allowed values:
- `ORIGINAL` – top‑level post/clip.
- `REPLY` – reply; `thread_ref` **MUST** be set to target `packet_id`.
- `REPOST` – pure share; `repost_of` **MUST** be set.
- `QUOTE` – share with commentary; `quote_ref` **MUST** be set.
- `ARTICLE` – news/URL‑centric; `article` SHOULD be present.
- `SHORT_VIDEO` – short‑form video; `short_video` SHOULD be present.
- `STORY` – ephemeral story (client‑enforced lifetime).

Group scoping:
- `group_ref` (optional) links a POST to a `GROUP` Packet (`packet_id`). Group feeds and filters SHOULD use `group_ref` as the canonical association; app_metadata MUST NOT be the primary mechanism for group membership context.

### 5.3 audience

Visibility intent (advisory; relays do not enforce):
- `PUBLIC`
- `FOLLOWERS`
- `CIRCLE`
- `UNLISTED`

Clients/services MAY enforce; relays treat packets as opaque.

### 5.4 link_previews

For posts with URLs:

```jsonc
"link_previews": [
  {
    "url": "https://news.example/story",
    "title": "Example passes open-data bill",
    "description": "Short summary...",
    "site_name": "Example News",
    "image": {
      "media_hash": "0x1e208f2c3a4b5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6",
      "media_type": "image/webp",
      "width": 1200,
      "height": 630
    }
  }
]
```

### 5.5 article (news / longform)

For `post_kind = "ARTICLE"`:

```jsonc
"article": {
  "headline": "City council passes open-data bill",
  "canonical_url": "https://news.example/story",
  "publisher_id": "did:strata:example_news",
  "byline": "Jane Doe",
  "section": "politics",
  "published_at": 1715421000,
  "summary": "One or two sentences...",
  "language": "en"
}
```

`publisher_id` MAY differ from `author_id`.

### 5.6 short_video (STRATAZ)

For `post_kind = "SHORT_VIDEO"`:

```jsonc
"short_video": {
  "loop": true,
  "max_duration_seconds": 60,
  "soundtrack": {
    "title": "Track Name",
    "artist": "Artist",
    "rights": "ugc-allowed"
  }
}
```

Video itself remains in `media`. STRATAZ SHOULD treat as vertical, swipeable, and surface provenance prominently.

## 6. PROFILE – user profile data

### 6.1 Type and structure

`content.type = "PROFILE"`:

```jsonc
{
  "type": "PROFILE",
  "profile_id": "did:strata:alice",
  "display_name": "Alice Example",
  "handle": "alice",
  "account_mode": "PUBLIC",
  "bio": "Bio text...",
  "location": "Berlin, DE",
  "avatar": {
    "media_hash": "0x9bcb25c9adc112b7cc9a93cae41f3262af1349b9f5f9a1a6a0404dea36dcc949",
    "media_type": "image/png"
  },
  "banner": {
    "media_hash": "0x7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e",
    "media_type": "image/jpeg"
  },
  "links": [
    { "title": "Website", "url": "https://alice.example" }
  ],
  "app_overrides": {
    "STRATAZ": { "featured_video": "0xPacketShortVideo..." }
  },
  "app_metadata": { ... }
}
```

Rules:
- `author_id` **MUST** equal `profile_id` for canonical self‑profiles.
- `account_mode` OPTIONAL. Valid values: `PUBLIC` (default) or `PROTECTED`.
- Third‑party profiles MAY use `profile_id ≠ author_id`; clients SHOULD treat separately from self‑authored profiles.
- For self-authored profiles (`author_id` equals `profile_id`), the latest valid PROFILE Packet by `received_at` MUST be treated as canonical for display purposes (display_name, avatar, handle, bio, etc.).
- Clients MAY maintain separate views for third-party profiles (`author_id` differs from `profile_id`), but these MUST NOT override the canonical self-profile in default UX.

### 6.2 Account mode: PROTECTED (optional)

`PROTECTED` indicates the author expects follow approvals and reduced visibility for non-approved users.

Clients that honor `PROTECTED`:
- SHOULD require follow approval before treating a follower as active.
- SHOULD hide followers/following lists from non-approved viewers.
- SHOULD default new posts to a followers-only audience where such a concept exists.
- MUST warn that this does not guarantee privacy unless relays enforce access control or content is encrypted.

### 6.3 Relay enforcement options (optional)

Relays MAY choose to enforce `PROTECTED` account visibility. This requires a viewer identity (AUTH) and a discoverable approval signal.

Options:
- **None (default):** Relay does not enforce. Clients handle visibility locally.
- **Soft enforcement:** Relay hides protected content from non-approved viewers but may still allow profile fetches.
- **Strict enforcement:** Relay denies protected content and profile access to non-approved viewers.

If a relay enforces `PROTECTED`:
- It MUST require client authentication to determine the viewer DID.
- It MUST define how approvals are evaluated (e.g., `FOLLOW_REQUEST` + `FOLLOW_APPROVAL` in `GLOBAL` or `APP:<APP_ID>` scope).
- It SHOULD advertise enforcement policy in its relay descriptor/manifest (RFC-0007).
- It SHOULD avoid federating protected content to peers that do not enforce equivalent policy.

Relays are not required to implement enforcement for interoperability.
Official Strata relays SHOULD use `strict` enforcement.

## 7. SOCIAL_EDGE and SOCIAL_EDGE_REVOCATION

Represents social relations (follows/blocks/mutes), distinct from TRUST_EDGE used for reputation.

### 7.1 SOCIAL_EDGE

```jsonc
{
  "type": "SOCIAL_EDGE",
  "from": "did:strata:alice",
  "to": "did:strata:bob",
  "relation": "FOLLOW",
  "scope": "GLOBAL",
  "created_at": 1715421000,
  "metadata": {
    "lists": ["close-friends"],
    "notes": "met at conference"
  },
  "app_metadata": { ... }
}
```

Rules:
- `author_id` **MUST** equal `from`.
- `relation` initial values: `FOLLOW`, `BLOCK`, `MUTE`, `CLOSE_FRIEND`.
- `BLOCK` and `MUTE` are privacy-sensitive; see §7.4 for client handling.
- Group membership MUST be expressed with `GROUP` + `GROUP_MEMBERSHIP` (see §9); `MEMBER` is not valid for SOCIAL_EDGE.
- `created_at` SHOULD mirror Packet timestamp; prefer `received_at` for ordering.

### 7.2 scope

Indicates applicability (normalized grammar):
- `GLOBAL` (default),
- `APP:<APP_ID>` where `<APP_ID>` is uppercase token `[A-Z0-9_]+` (e.g., `APP:STRATAX`, `APP:STRATAZ`, `APP:STRATAF`, `APP:CTRL+F5` → `APP:CTRL+F5` becomes `APP:CTRL_F5` for normalization),
- `GROUP:<packet_id>` referencing a `GROUP` Packet.

Unknown or nonconformant scopes **MUST** be ignored.

### 7.3 SOCIAL_EDGE_REVOCATION

```jsonc
{
  "type": "SOCIAL_EDGE_REVOCATION",
  "from": "did:strata:alice",
  "to": "did:strata:bob",
  "relation": "FOLLOW",
  "scope": "GLOBAL",
  "revokes": ["0xEdgePacketId1", "0xEdgePacketId2"],
  "reason": "user_unfollowed",
  "created_at": 1715422000
}
```

Rules:
- `author_id` **MUST** equal `from`.
- Sets effective presence of `(from, to, relation, scope)` to absent for all SOCIAL_EDGE with `created_at <= revocation.created_at`.
- `revokes` OPTIONAL; points to specific SOCIAL_EDGE `packet_id`s.
- Clients SHOULD keep history but compute an effective graph per `(from, to, relation, scope)`.

### 7.4 Privacy-sensitive relations (BLOCK / MUTE)

`BLOCK` and `MUTE` are sensitive relationship signals and SHOULD NOT be published by default.

Guidance:
- Clients SHOULD treat BLOCK/MUTE as local preference state unless the user explicitly opts in to publishing them.
- Clients MAY offer an opt-in "publish block signal" option; if enabled, clients MUST warn that the relation is public and queryable.
- Clients MUST NOT assume network visibility for BLOCK/MUTE; absence of a public edge does not imply absence of a local block/mute.
- Relays and indexers MUST continue to treat BLOCK/MUTE edges as ordinary SOCIAL_EDGE packets when they are published.

### 7.5 Follow requests for protected accounts (optional)

When a profile declares `account_mode = "PROTECTED"`, clients SHOULD use follow requests.

`relation` values:
- `FOLLOW_REQUEST` indicates a pending request to follow a protected account.
- `FOLLOW_APPROVAL` indicates approval by the protected account for a requester.

Guidance:
- A requester SHOULD publish `FOLLOW_REQUEST` from requester → protected account.
- The protected account SHOULD publish `FOLLOW_APPROVAL` from protected account → requester.
- Clients SHOULD treat a follower as active only when a non-revoked `FOLLOW_REQUEST` and `FOLLOW_APPROVAL` exist for the same `(requester, protected)` pair.
- To preserve existing followers when switching to `PROTECTED`, clients and relays MAY treat a non-revoked `FOLLOW` edge that predates the switch as an implicit approval.
- For public accounts, clients SHOULD use `FOLLOW` directly, not `FOLLOW_REQUEST`.

This is a social-layer signal only; relays are not required to enforce visibility or access control. If approvals are kept local-only, relays cannot enforce PROTECTED visibility.

## 8. REACTION – likes and emoji reactions

### 8.1 Type and structure

```jsonc
{
  "type": "REACTION",
  "from": "did:strata:alice",
  "target_packet": "0xPostPacketId",
  "reaction_kind": "LIKE",
  "emoji": "🔥",
  "created_at": 1715421300,
  "app_metadata": { ... }
}
```

Rules:
- `author_id` **MUST** equal `from`.
- `reaction_kind` initial values: `LIKE`, `DISLIKE`, `EMOJI`, `BOOKMARK`, `VOTE_UP`, `VOTE_DOWN`.
- For `EMOJI`, `emoji` SHOULD be set.
- For aggregates, clients SHOULD rely on `CONSENSUS_METRIC` Packets (RFC‑0002 §7), not on-the-fly counts.

### 8.2 Idempotency

Clients SHOULD ensure at most one active reaction per `(from, target_packet, reaction_kind)`:
- To change/undo: emit new REACTION and let aggregators pick latest by `received_at`, or
- Emit a `CORRECTION` targeting the old reaction.

## 9. GROUP and GROUP_MEMBERSHIP

### 9.1 GROUP

```jsonc
{
  "type": "GROUP",
  "group_id": "0xGroupPacketId",
  "name": "Strata Devs",
  "description": "Developers working on Strata.",
  "visibility": "PUBLIC",
  "open_membership": false,
  "avatar": {
    "media_hash": "0x36dcc9499bcb25c9adc112b7cc9a93cae41f3262af1349b9f5f9a1a6a0404dea",
    "media_type": "image/png"
  },
  "banner": {
    "media_hash": "0x3a4b5d6e7f8091a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6f708192a",
    "media_type": "image/jpeg"
  },
  "rules": "Markdown or text...",
  "app_metadata": { ... }
}
```

Fields:
- `author_id` is creator/owner.
- `group_id` SHOULD equal this Packet's `packet_id`.
- `open_membership` - boolean indicating whether users can self-join without authorization. Default `false` if omitted.

### 9.2 GROUP_MEMBERSHIP

```jsonc
{
  "type": "GROUP_MEMBERSHIP",
  "group_id": "0xGroupPacketId",
  "member_id": "did:strata:alice",
  "role": "MEMBER",
  "action": "ADD",
  "created_at": 1715421400,
  "app_metadata": { ... }
}
```

Fields:
- `role` initial values: `OWNER`, `ADMIN`, `MODERATOR`, `MEMBER`.
- `action`: `ADD` | `REMOVE`.

**Authority Set:**

The *authority set* for a group determines who may modify membership:

- The `author_id` of the GROUP Packet is the initial OWNER and is automatically in the authority set.
- The authority set consists of all identities with `role` equal to `OWNER` or `ADMIN` in the current effective membership view (computed by ordering GROUP_MEMBERSHIP Packets by `received_at`).

**Authorization Rules:**

- **Self-actions** (`author_id` equals `member_id`):
  - `action: "ADD"` (self-join): MUST be ignored unless the GROUP has `open_membership = true`, OR `author_id` is already in the authority set.
  - `action: "REMOVE"` (self-leave): Always allowed for any current member.

- **Non-self actions** (`author_id` differs from `member_id`):
  - MUST be ignored unless `author_id` is in the group's authority set at the time of the Packet's `received_at`.
  - Only authorities may add members, remove members, or change roles.

- **Role changes:**
  - Only an OWNER may promote another member to OWNER or ADMIN.
  - Only an OWNER or ADMIN may demote or remove members with role MEMBER or MODERATOR.
  - An OWNER cannot be removed except by self-leave or by another OWNER.

**Validation:**

Clients MUST compute the effective membership by:
1. Starting with the GROUP Packet's `author_id` as OWNER.
2. Processing GROUP_MEMBERSHIP Packets in `received_at` order.
3. For each Packet, checking authorization rules above before applying the membership change.
4. Ignoring Packets that fail authorization checks.

## 10. LIST and LIST_MEMBERSHIP – user-curated lists

User-curated lists allow users to organize other users into named collections for custom timeline viewing (e.g., "Photographers", "News Sources").

### 10.1 LIST

Defines a user-curated list.

```jsonc
{
  "type": "LIST",
  "list_id": "0xListPacketId",
  "name": "Photographers",
  "description": "My favorite photographers on Strata.",
  "visibility": "PUBLIC",
  "created_at": 1715421500,
  "app_metadata": { ... }
}
```

Fields:
- `author_id` is the list owner.
- `list_id` SHOULD equal this Packet's `packet_id`.
- `visibility`: `PUBLIC` | `PRIVATE`. Private lists are only visible to the owner.
- `name` (required): Display name for the list.
- `description` (optional): Description text.
- `created_at`: Unix timestamp of list creation.

**Updates:** To update list metadata (name, description, visibility), emit a new LIST Packet with the same `list_id`; clients SHOULD use the latest by `received_at`.

**Deletion:** Use CORRECTION with `action: "retract"` targeting the LIST Packet.

### 10.2 LIST_MEMBERSHIP

Adds a user to a list.

```jsonc
{
  "type": "LIST_MEMBERSHIP",
  "list_id": "0xListPacketId",
  "member_did": "did:strata:bob",
  "added_at": 1715421600,
  "app_metadata": { ... }
}
```

Fields:
- `author_id` **MUST** equal the list owner (the `author_id` of the referenced LIST Packet).
- `list_id`: References the LIST Packet's `list_id`.
- `member_did`: The DID being added to the list.
- `added_at`: Unix timestamp of addition.

**Removal:** Use CORRECTION with `action: "retract"` targeting the LIST_MEMBERSHIP Packet.

**Authorization:** Only the list owner may emit LIST_MEMBERSHIP Packets for their lists; Packets from other authors MUST be ignored.

**Validation:** Clients compute effective list membership by:
1. Finding the latest LIST Packet for a given `list_id` by `received_at`.
2. Processing LIST_MEMBERSHIP Packets in `received_at` order.
3. Checking that `author_id` matches the list owner before applying membership.
4. Ignoring Packets that fail authorization or have been retracted via CORRECTION.

## 11. Interactions with trust, attestations, and Reality Switch

- TRUST_EDGE / TRUST_REVOCATION (RFC‑0001) remain distinct from SOCIAL_EDGE.
- Attestations targeting POST (e.g., `MANIPULATED`, `OUT_OF_CONTEXT`) SHOULD be surfaced across apps and fed into Reality Switch (RFC‑0005).
- SOCIAL_EDGE relations may hint Web‑of‑Trust inputs but SHOULD NOT replace explicit TRUST_EDGEs for reputation.

## 12. App mappings (non-normative)

### 12.1 STRATAX (X-style microblog)
- Uses POST (`ORIGINAL`, `REPLY`, `REPOST`, `QUOTE`), PROFILE, SOCIAL_EDGE/REVOCATION (follows/blocks/mutes), REACTION (likes), MESSAGE (RFC‑0002), LIST + LIST_MEMBERSHIP for curated timelines.
- Follows: SOCIAL_EDGE `relation = "FOLLOW"`, `scope = "APP:STRATAX"` or `GLOBAL`.

### 12.2 STRATAZ (short-form video)
- Uses POST with `post_kind = "SHORT_VIDEO"`, PROFILE, SOCIAL_EDGE (follows), REACTION (likes/emoji); emphasizes provenance/attestations.

### 12.3 STRATAF (Facebook-style)
- Uses POST (`ORIGINAL`, `ARTICLE`, `STORY`), PROFILE, SOCIAL_EDGE (friends/follows), GROUP + GROUP_MEMBERSHIP for communities; feeds filter POST by group context.

### 12.4 CTRL+F5 (news hub)
- Uses POST (`ARTICLE` + `article` payload), PROFILE for publishers/journalists, REACTION for reader votes, attestations/retroactive consensus (RFC‑0003) for labeling.

## 13. Backwards compatibility

- POST remains valid with minimal text/media/thread_ref per RFC‑0002.
- Clients not understanding new fields/types MUST still verify signatures and present fallback UI.
- Future RFCs MAY add new `post_kind`, `relation`, `scope`, or new social types (e.g., EVENT, POLL); unknown values MUST NOT cause hard failures.
