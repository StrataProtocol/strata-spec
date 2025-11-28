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
- Introduces `PROFILE`, `SOCIAL_EDGE`, `SOCIAL_EDGE_REVOCATION`, `REACTION`, `GROUP`, `GROUP_MEMBERSHIP`,
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
  "media_hash": "ipfs://Qm...",
  "media_type": "video/mp4",
  "size_bytes": 123456,
  "duration_seconds": 12.3,
  "width": 1080,
  "height": 1920,
  "variants": [
    {
      "label": "thumb",
      "media_hash": "ipfs://QmThumb...",
      "media_type": "image/webp",
      "width": 320,
      "height": 568
    },
    {
      "label": "720p",
      "media_hash": "ipfs://Qm720p...",
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
      "media_hash": "ipfs://QmPreview...",
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
  "bio": "Bio text...",
  "location": "Berlin, DE",
  "avatar": {
    "media_hash": "ipfs://QmAvatar...",
    "media_type": "image/png"
  },
  "banner": {
    "media_hash": "ipfs://QmBanner...",
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
- Third‑party profiles MAY use `profile_id ≠ author_id`; clients SHOULD treat separately from self‑authored profiles.
- For self-authored profiles (`author_id` equals `profile_id`), the latest valid PROFILE Packet by `received_at` MUST be treated as canonical for display purposes (display_name, avatar, handle, bio, etc.).
- Clients MAY maintain separate views for third-party profiles (`author_id` differs from `profile_id`), but these MUST NOT override the canonical self-profile in default UX.

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
    "media_hash": "ipfs://QmGroupAvatar...",
    "media_type": "image/png"
  },
  "banner": {
    "media_hash": "ipfs://QmGroupBanner...",
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

## 10. Interactions with trust, attestations, and Reality Switch

- TRUST_EDGE / TRUST_REVOCATION (RFC‑0001) remain distinct from SOCIAL_EDGE.
- Attestations targeting POST (e.g., `MANIPULATED`, `OUT_OF_CONTEXT`) SHOULD be surfaced across apps and fed into Reality Switch (RFC‑0005).
- SOCIAL_EDGE relations may hint Web‑of‑Trust inputs but SHOULD NOT replace explicit TRUST_EDGEs for reputation.

## 11. App mappings (non-normative)

### 11.1 STRATAX (X-style microblog)
- Uses POST (`ORIGINAL`, `REPLY`, `REPOST`, `QUOTE`), PROFILE, SOCIAL_EDGE/REVOCATION (follows/blocks/mutes), REACTION (likes), MESSAGE (RFC‑0002).
- Follows: SOCIAL_EDGE `relation = "FOLLOW"`, `scope = "APP:STRATAX"` or `GLOBAL`.

### 11.2 STRATAZ (short-form video)
- Uses POST with `post_kind = "SHORT_VIDEO"`, PROFILE, SOCIAL_EDGE (follows), REACTION (likes/emoji); emphasizes provenance/attestations.

### 11.3 STRATAF (Facebook-style)
- Uses POST (`ORIGINAL`, `ARTICLE`, `STORY`), PROFILE, SOCIAL_EDGE (friends/follows), GROUP + GROUP_MEMBERSHIP for communities; feeds filter POST by group context.

### 11.4 CTRL+F5 (news hub)
- Uses POST (`ARTICLE` + `article` payload), PROFILE for publishers/journalists, REACTION for reader votes, attestations/retroactive consensus (RFC‑0003) for labeling.

## 12. Backwards compatibility

- POST remains valid with minimal text/media/thread_ref per RFC‑0002.
- Clients not understanding new fields/types MUST still verify signatures and present fallback UI.
- Future RFCs MAY add new `post_kind`, `relation`, `scope`, or new social types (e.g., EVENT, POLL); unknown values MUST NOT cause hard failures.
