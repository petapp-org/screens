# Screen 1: Home / Explore

## Overview

The Explore screen is the **default landing screen** for all users — authenticated or not.  
On app launch / web root (`/`), users are always redirected here.  
Default active filter tab: **Latest**.

---

## UI Layout

```
[Header]
  Title: "Explore"
  Right actions: [Search icon] | [Messages icon ✉] | [Notifications icon 🔔] | [Profile avatar]

[Filter Tabs]
  Latest | Follow | Rescue

[Feed — infinite scroll]
  Post Card 1
  Post Card 2
  ...
  Post Card N  ← after the 1st post, inject "Suggested Families" widget
  ...

[Bottom Navigation]
  My Pets | Explore (active) | Shops | Services | More

  Auth rules:
  - Explore: accessible without login (default landing screen)
  - My Pets, Shops, Services, More: redirect to Login if not authenticated
```

---

## Components

### 0. Header Buttons

| Button | Logged-in | Not logged-in |
|--------|-----------|---------------|
| Search | Opens search screen | Tapping redirects to Login (auth required — see screen_11) |
| Messages `✉` | Opens the **Messages screen** (screen_10); shows a **red dot** when there is any unread chat | Tapping redirects to Login; **no red dot shown** |
| Notifications `🔔` | Opens the **Notifications screen** (screen_22); shows a **red dot** when there is any unread activity | Tapping redirects to Login; **no red dot shown** |
| Profile | Shows user's avatar; tapping opens profile screen | Shows default placeholder avatar; tapping redirects to Login |

The unread state is fetched separately (not bundled in the feed response) — the Messages dot from `UnreadMessageCount (BL)` (see screen_10), the Notifications dot from `UnreadNotificationCount (BU)` (see screen_22); each delivered via WebSocket push, falling back to polling. Each dot shows whenever its own count `> 0` (no number).

---

### 1. Filter Tabs

3 fixed tabs — no dynamic categories.

| Tab | `filter` (ExploreFilter) | Description | Auth required |
|-----|-------------------------|-------------|---------------|
| Latest | `ALL` | Newest posts across all families, sorted by `createdAt` desc | No |
| Follow | `FOLLOW` | Posts from families the current user follows | Yes — redirect to Login if not authenticated |
| Rescue | `RESCUE` | Posts from families with `familyType = CHARITY` | No |

**Inputs:**
- `sort`: `ExploreSort` — `LATEST` (default) | `TOP`; current tabs always use `LATEST`
- `filter`: `ExploreFilter` — `ALL` | `FOLLOW` | `RESCUE` (tab mapping above)
- `after`: opaque pagination cursor — `pageInfo.endCursor` of the previous page (absent on first load)
- `first`: page size, default `20`

---

### 2. Post Card

> **Canonical definition.** This post card layout applies to ALL screens that display posts: Explore feed, Post Detail, Family Posts, Pet Posts, Random Pet Posts, User Posts, and Loved Posts. Any update here propagates to all screens.

Each post card displays:

| Field | Description |
|-------|-------------|
| `avatarUrl` | Display avatar: always use family avatar |
| `familyName` | Name of the family that created the post |
| `authorName` | Display name of the user who created the post. Shown top-right: `"authorName · time"` |
| `pets` | List of **named pets** linked across all media in this post (only media with `mediaTag.type = PET`). Used to render subtitle e.g. `"Pudding · Mochi"`. Each item: `{ id, name, avatarUrl }`. Can be empty. |
| `body` | Post text / description |
| `location` | Optional. `{ "city": "Hồ Chí Minh", "cityCode": "HCM", "country": "Việt Nam", "countryCode": "VN" }`. Shown bottom-right below author/time line: `"HCM - VN"`. Omitted if `null`. |
| `media` | List of media items (see Media Object below). **Minimum 1, maximum 10.** |
| `loveCount` | Total number of loves |
| `commentsCount` | Total number of comments |
| `createdAt` | ISO 8601 timestamp |
| `isLoved` | Boolean — whether the current user has loved this post (false if not logged in) |
| `visibility` | `PUBLIC` \| `FOLLOWERS` \| `PRIVATE` — see visibility rules below |

**PostMedia Object (`Post.media: [PostMedia!]`):**

```json
{
  "order": "int",
  "sourceType": "UPLOADED | EMBEDDED",
  "embedUrl": "string | null",
  "embedProvider": "YOUTUBE | VIMEO | null",
  "mediaTag": {
    "type": "PET | RANDOM",
    "petId": "string | null",
    "species": "string | null",
    "breed": "string | null"
  },
  "media": {
    "id": "string",
    "type": "IMAGE | VIDEO | DOCUMENT",
    "thumbnailUrl": "string | null",
    "variants": [],
    "hlsUrl": "string | null",
    "blurhash": "string | null",
    "signedUrl": "string",
    "expiresAt": "string | null",
    "mimeType": "image/jpeg",
    "width": 1080,
    "height": 1350,
    "durationSeconds": null
  }
}
```

**`mediaTag` types:**
| Type | Meaning | `petId` | `species` | `breed` |
|------|---------|---------|-----------|---------|
| `PET` | AI matched to a named pet in the family | `petId` — matches an entry in `post.pets` | species of the matched pet | breed of the matched pet |
| `RANDOM` | AI detected no named pet match, or media is `EMBEDDED` | `null` | species string if AI detected an animal (e.g. `"cat"`), or `null` | breed string if AI identified a specific breed, or `null` |

**Rendering rules:**
- `UPLOADED` + `media.type = IMAGE` → use `media.signedUrl` as the displayable URL
- `UPLOADED` + `media.type = VIDEO` → play via **`media.hlsUrl`** (HLS playlist), **not** `signedUrl` — video `signedUrl` trỏ source object private và trả 403 (petapp-be#917 / PR #975). Dùng `media.thumbnailUrl` làm poster trong khi HLS load.
- `EMBEDDED` → display embed player (YouTube/Vimeo); use `media.thumbnailUrl` as poster image; `embedUrl` is the external video URL
- Multiple media items → swipeable carousel with `N/Total` indicator (e.g. `1/3`) in the top-right corner of the media area

**Pet badge (bottom-left of media):**
- `mediaTag.type = PET` → show floating badge: `[pet avatar]  pet name`; look up display info using `mediaTag.petId` against `post.pets` (where `pets[].id = mediaTag.petId`); tappable → Pet Posts screen
- `mediaTag.type = RANDOM` → **no badge shown** (regardless of whether `breed`/`species` is populated; breed/species shown in Random Pets section context only)
- Each media item in the carousel evaluates independently

**Private pet enforcement (server-side):**
- If `pet.isPublic = false` AND viewer is not a family member → server returns `mediaTag` as `{ type: RANDOM, species: ..., breed: ... }` instead of `{ type: PET, petId: ..., ... }`
- Client applies existing rules: `type = RANDOM` → no badge; no special client logic needed

**Post Card Footer:**

```
287 loves · 34 comments    [Love icon]  Love    [Comment icon]  Comment
```

- **Love button:**
  - Tapping toggles love state (requires login — redirect to Login if not authenticated)
  - **Optimistic update**: `loveCount` and `isLoved` are updated immediately in the UI before the API response returns
  - On API error: revert `loveCount` and `isLoved` back to previous values and show error toast
  - `287 loves` text is tappable → same behaviour as Love button

- **Comment button / count:**
  - `34 comments` text and Comment button are both tappable
  - Tapping opens an **inline comment panel** (expands below the post card, does not navigate away)
  - Inline panel shows the **last 10 comments**, sorted by `createdAt` asc (oldest first within the set); panel **auto-scrolls to bottom** on open so the newest comment is visible
  - If `commentsCount > 10`: show a "View all N comments" link → navigates to Post Detail screen
  - User can submit a new comment directly from the inline panel (requires login)
  - **Close:** tap anywhere outside the panel (on the feed) to collapse it

- Tapping the **post media or caption area** → opens Post Detail screen (full-screen view with all comments)
- Tapping **family name** → My Pets screen if the current user is a member of that family; otherwise Family Posts screen
- Tapping **author name** → User Posts screen (all posts by that user)
- Tapping **pet badge** on media → Pet Posts screen (all posts linked to that pet)

**Post Visibility Rules (enforced server-side — client never receives posts it shouldn't see):**

| Privacy value | Who can see |
|---------------|-------------|
| `public` | Everyone, including unauthenticated users |
| `followers` | Family members of the authoring family + users who follow that family |
| `private` | Family members of the authoring family only |

Unauthenticated requests → server returns `public` posts only.

---

### 3. Post Context Menu (`...` button)

**Case A — Logged-in user is the post author OR a member of the post's family:**
- Edit post
- Delete post
- Change privacy (`public` / `followers only` / `private`)

**Case B — Logged-in user is a different user:**
- Hide this post
- Hide all posts from this user
- Hide all posts from this family

**Case C — User is not logged in:**
- Tapping `...` → redirect to Login screen

---

### 4. Suggested Families Widget

Injected **after the 1st post** in the feed. Persists in position on scroll; refreshes on page/feed reload.

| Field | Description | Displayed for |
|-------|-------------|---------------|
| `familyId` | Unique ID | all |
| `familyName` | Display name | all |
| `avatarUrl` | Family avatar | all |
| `social.followersCount` | Raw number | all |
| `shortDescription` | Free-text description set by the family | **charity only** — hidden for standard families |
| `familyType` | `NORMAL` \| `CHARITY` | all (drives UI logic) |
| `social.isFollowedByMe` | Boolean | all |

> followersCount trả số thô; client tự format "3.6k" theo locale (bỏ followerCountDisplay — i18n client-side).

**Buttons per family:**
- `NORMAL` type → **Follow** button only
- `CHARITY` type → **Follow** button + **Donate** button

**Donate button behaviour:** (interim — no wallet yet)
- Tapping Donate → **opens a chat** with that charity family, pre-filled with a Vietnamese support message (canonical behaviour defined in `screen_3` → Charity Section → Donate button behaviour; mechanic identical to Lost Pet "I saw").
- Not logged in → redirect to Login. Hidden for members of that charity.
- No external URL needed; navigation is handled in-app by family ID

**Dismiss / Hide:**
- Each family row has an `×` (dismiss) button
- Tapping `×` marks that family as "dismissed" for this session
- Dismissed families are excluded from future suggestions **within this session**
- On next page reload / app restart, a new set of 5 suggestions is shown (excluding any permanently dismissed ones if the user is logged in)
- If user is logged in: dismissal is persisted server-side (never suggest this family again until user clears history)
- If user is not logged in: dismissal is session-only (localStorage / in-memory)

**Inputs (API):**
- `first`: page size, fixed at `5` for the widget (server may return fewer)
- `after`: pagination cursor (omit on first load); server already excludes dismissed families (via `DismissFamilySuggestion C`) and already-followed families for authenticated callers — client does not need to pass `excludeIds`

---

## API Endpoints Required

All API calls go to a single `POST /graphql` endpoint.
**Auth header:** `Authorization: Bearer <token>` (optional unless stated otherwise).

---

### A. Query: `ExploreFeed`

Fetches the paginated post feed for the Explore screen.

**Auth:** Optional. When present, `isLoved` and viewer-specific fields are populated.

**Operation:**
```graphql
query ExploreFeed($sort: ExploreSort! = LATEST, $filter: ExploreFilter! = ALL, $first: Int! = 20, $after: String) {
  exploreFeed(sort: $sort, filter: $filter, first: $first, after: $after) {
    edges {
      node {
        id
        family {
          id
          name
          avatarUrl
          familyType
        }
        author {
          id
          displayName
          avatarUrl
        }
        pets {
          id
          name
          avatarUrl
        }
        body
        location {
          city
          cityCode
          country
          countryCode
        }
        media {
          order
          sourceType
          embedUrl
          embedProvider
          mediaTag {
            type
            petId
            species
            breed
          }
          media {
            id
            type
            thumbnailUrl
            variants { size key contentType }
            hlsUrl
            blurhash
            signedUrl
            expiresAt
          }
        }
        loveCount
        commentsCount
        isLoved
        visibility
        createdAt
      }
    }
    pageInfo {
      endCursor
      hasNextPage
    }
  }
}
```

**Variables:**
```json
{ "sort": "LATEST", "filter": "ALL", "first": 10, "after": null }
```

**Response `200 OK`:**
```json
{
  "data": {
    "exploreFeed": {
      "edges": [
        {
          "node": {
            "id": "post_abc123",
            "family": {
              "id": "fam_xyz",
              "name": "Pudding's Family",
              "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
              "familyType": "NORMAL"
            },
            "author": {
              "id": "user_001",
              "displayName": "Mochi",
              "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
            },
            "pets": [
              {
                "id": "pet_111",
                "name": "Pudding",
                "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg"
              },
              {
                "id": "pet_222",
                "name": "Mochi",
                "avatarUrl": "https://cdn.petapp.com/pets/pet_222/avatar.jpg"
              }
            ],
            "body": "Pudding nằm chờ mama nấu cơm 🌕 Q7 cat life",
            "location": {
              "city": "Hồ Chí Minh",
              "cityCode": "HCM",
              "country": "Việt Nam",
              "countryCode": "VN"
            },
            "media": [
              {
                "order": 1,
                "sourceType": "UPLOADED",
                "embedUrl": null,
                "embedProvider": null,
                "mediaTag": {
                  "type": "PET",
                  "petId": "pet_111",
                  "species": "Cat",
                  "breed": "Orange Tabby Cat"
                },
                "media": {
                  "id": "media_001",
                  "type": "IMAGE",
                  "thumbnailUrl": null,
                  "variants": [{ "size": "1080x1080", "key": "media/001_1080.jpg", "contentType": "image/jpeg" }],
                  "hlsUrl": null,
                  "blurhash": "L6PZfSi_.AyE_3t7t7R**0o#DgR4",
                  "signedUrl": "https://cdn.petapp.com/media/001.jpg?token=abc",
                  "expiresAt": "2026-06-13T06:00:00Z"
                }
              },
              {
                "order": 2,
                "sourceType": "EMBEDDED",
                "embedUrl": "https://www.youtube.com/watch?v=abc123",
                "embedProvider": "YOUTUBE",
                "mediaTag": {
                  "type": "RANDOM",
                  "petId": null,
                  "species": null,
                  "breed": null
                },
                "media": null
              },
              {
                "order": 3,
                "sourceType": "UPLOADED",
                "embedUrl": null,
                "embedProvider": null,
                "mediaTag": {
                  "type": "RANDOM",
                  "petId": null,
                  "species": "Cat",
                  "breed": "British Shorthair"
                },
                "media": {
                  "id": "media_003",
                  "type": "IMAGE",
                  "thumbnailUrl": null,
                  "variants": [{ "size": "1080x1080", "key": "media/003_1080.jpg", "contentType": "image/jpeg" }],
                  "hlsUrl": null,
                  "blurhash": "L5H2EC=PM+yV0g-mq.wG9c010J}I",
                  "signedUrl": "https://cdn.petapp.com/media/003.jpg?token=def",
                  "expiresAt": "2026-06-13T06:00:00Z"
                }
              }
            ],
            "loveCount": 287,
            "commentsCount": 34,
            "isLoved": false,
            "visibility": "PUBLIC",
            "createdAt": "2026-06-06T06:00:00Z"
          }
        }
      ],
      "pageInfo": {
        "endCursor": "eyJpZCI6InBvc3RfYWJjMTIzIn0=",
        "hasNextPage": true
      }
    }
  }
}
```

> Example above: post has 3 media. Media 1 (uploaded image) → linked to Pudding. Media 2 (embedded YouTube) → random. Media 3 (uploaded image) → no pet linked, no badge shown.

**Notes:**
- `media` list has minimum 1, maximum 10 items
- `pets` contains only named pets (`mediaTag.type = PET`), deduplicated; can be empty `[]`
- Only `mediaTag.type = PET` items contribute to `post.pets` list; `RANDOM` media are NOT shown in subtitle
- Post header layout:
  ```
  [family avatar]  Family Name          displayName · 3h
                   Pet1 · Pet2          HCM - VN
  ```
  - Top-left: family name
  - Bottom-left: pet names joined by ` · ` (omit row if `pets` is empty)
  - Top-right: `displayName · time` (e.g. `"Mochi · 3h"`) — time format follows the rules below:

**Time display rules (based on `createdAt`):**

| Condition | Format | Example |
|-----------|--------|---------|
| < 1 minute ago | `just now` | `Mochi · just now` |
| < 1 hour ago | `Xm` | `Mochi · 5m` |
| < 24 hours ago | `Xh` | `Mochi · 3h` |
| < 7 days ago | `Xd` | `Mochi · 2d` |
| ≥ 7 days ago, same year | `DD MMM` | `Mochi · 28 May` |
| Different year | `DD/MM/YYYY` | `Mochi · 15/03/2025` |

- On web: hovering the time text shows a tooltip with the full datetime (e.g. `"06/06/2026 13:00"`)
  - Bottom-right: location as `cityCode - countryCode` (omit if `location` is null)
- `sort`: `ExploreSort` — `LATEST` (newest first, default) | `TOP` (highest engagement first)
- `filter: FOLLOW` requires authentication → returns GraphQL error with code `UNAUTHORIZED` if no valid token
- `filter: RESCUE` returns posts from families where `family.familyType = CHARITY`
- `isLoved` is always `false` when unauthenticated
- Server enforces privacy rules before returning results — unauthenticated callers only receive `PUBLIC` posts; `FOLLOWERS` posts are filtered based on the caller's follow list

**Errors:**

| Code | Scenario |
|------|----------|
| `INVALID_FILTER` | Unknown filter value |
| `UNAUTHORIZED` | `filter: FOLLOW` without auth token |
| `INVALID_CURSOR` | Cursor is malformed or expired |

---

### B. Query: `SuggestedFamilies`

Returns families to show in the Suggested Families widget. Shipped petapp-be#947.

**Auth:** Optional. When authenticated, the server automatically excludes families the caller already follows and any families the caller has dismissed via `DismissFamilySuggestion (C)` — no need to pass `excludeIds` client-side.

**Operation:**
```graphql
query SuggestedFamilies($first: Int! = 20, $after: String) {
  suggestedFamilies(first: $first, after: $after) {
    edges {
      cursor
      node {
        id
        name
        avatarUrl
        social {
          followersCount
          isFollowedByMe
        }
        shortDescription
        familyType
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Variables:**
```json
{ "first": 5, "after": null }
```

**Response `200 OK`:**
```json
{
  "data": {
    "suggestedFamilies": {
      "edges": [
        {
          "cursor": "eyJpZCI6ImZhbV9jYXRfaG91c2UifQ==",
          "node": {
            "id": "fam_cat_house",
            "name": "My's Cat House",
            "avatarUrl": "https://cdn.petapp.com/families/fam_cat_house/avatar.jpg",
            "social": {
              "followersCount": 3600,
              "isFollowedByMe": false
            },
            "shortDescription": "Rescue & rehome cats in HCM City",
            "familyType": "CHARITY"
          }
        },
        {
          "cursor": "eyJpZCI6ImZhbV9ub3JtYWxfMDAxIn0=",
          "node": {
            "id": "fam_normal_001",
            "name": "Mochi's Family",
            "avatarUrl": "https://cdn.petapp.com/families/fam_normal_001/avatar.jpg",
            "social": {
              "followersCount": 420,
              "isFollowedByMe": false
            },
            "shortDescription": null,
            "familyType": "NORMAL"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": false,
        "endCursor": "eyJpZCI6ImZhbV9ub3JtYWxfMDAxIn0="
      }
    }
  }
}
```

**Note:** `shortDescription` is always present in the response but is `null` for `NORMAL` families. The client should only render the description line when `familyType = CHARITY` and `shortDescription` is non-null.

`shortDescription` is a free-text field that the charity family admin fills in manually (via their family profile settings). It is not computed or auto-generated.

> `shortDescription` on Family: ⏳ GAP petapp-be#773 — field not yet in the schema; selection will be dropped at codegen until the backend ships it. Keep the documentation here for UX reference.

---

### C. Mutation: `DismissFamilySuggestion`

Marks a family as dismissed from suggestions for the authenticated user.

**Auth:** Required (for unauthenticated users, dismissal is handled client-side only)

**Operation:**
```graphql
mutation DismissFamilySuggestion($familyId: ID!) {
  dismissFamilySuggestion(familyId: $familyId)
}
```

**Variables:**
```json
{ "familyId": "fam_cat_house" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "dismissFamilySuggestion": true
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in (server-side dismissal only; client-only dismissal always succeeds) |
| `FAMILY_NOT_FOUND` | Family does not exist |

---

### D. Mutation: `FollowFamily`

Follow a family.

> **Result shape (ADR-0025):** the `FollowFamily`/`UnfollowFamily`/`LovePost`/`UnlovePost` mutations return the **affected entity** — select `family { social { … } }` (follow) or `post { isLoved loveCount }` (love) and read the new state from it (normalized cache updates, no refetch needed). The legacy flat fields (`familyId`/`alreadyFollowing`/`wasFollowing` on follow, `postId`/`alreadyLoved`/`wasLoved` on love) are `@deprecated` and will be removed in R+1.

> **Triggers notification:** fires a `NEW_FOLLOWER` notification to the followed family's members (see screen_22 — Notifications screen). Unfollowing (E) does not notify.

**Auth:** Required → returns GraphQL error with code `UNAUTHORIZED` if not logged in

**Operation:**
```graphql
mutation FollowFamily($familyId: ID!) {
  followFamily(familyId: $familyId) {
    family {
      social {
        isFollowedByMe
        followersCount
      }
    }
  }
}
```

**Variables:**
```json
{ "familyId": "fam_cat_house" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "followFamily": {
      "family": {
        "social": {
          "isFollowedByMe": true,
          "followersCount": 3601
        }
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `FAMILY_NOT_FOUND` | Family does not exist |
| `ALREADY_FOLLOWING` | Caller already follows this family |

---

### E. Mutation: `UnfollowFamily`

Unfollow a family.

**Auth:** Required

**Operation:**
```graphql
mutation UnfollowFamily($familyId: ID!) {
  unfollowFamily(familyId: $familyId) {
    family {
      social {
        isFollowedByMe
        followersCount
      }
    }
  }
}
```

**Variables:**
```json
{ "familyId": "fam_cat_house" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "unfollowFamily": {
      "family": {
        "social": {
          "isFollowedByMe": false,
          "followersCount": 3600
        }
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `FAMILY_NOT_FOUND` | Family does not exist |
| `NOT_FOLLOWING` | Caller is not following this family |

---

### F. Mutation: `LovePost`

Love a post.

> **Triggers notification:** fires a `POST_LOVES` notification to the post author — **grouped** per post (`{actor} and {N} others loved your post`) (see screen_22 — Notifications screen). Un-loving (G) does not notify.

**Auth:** Required → redirect to login if not authenticated

**Operation:**
```graphql
mutation LovePost($postId: ID!) {
  lovePost(postId: $postId) {
    post {
      isLoved
      loveCount
    }
  }
}
```

**Variables:**
```json
{ "postId": "post_abc123" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "lovePost": {
      "post": {
        "isLoved": true,
        "loveCount": 288
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `POST_NOT_FOUND` | Post does not exist or has been deleted |

---

### G. Mutation: `UnlovePost`

Un-love a post.

**Auth:** Required

**Operation:**
```graphql
mutation UnlovePost($postId: ID!) {
  unlovePost(postId: $postId) {
    post {
      isLoved
      loveCount
    }
  }
}
```

**Variables:**
```json
{ "postId": "post_abc123" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "unlovePost": {
      "post": {
        "isLoved": false,
        "loveCount": 287
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `POST_NOT_FOUND` | Post does not exist or has been deleted |
| `NOT_LOVED` | Caller has not loved this post |

---

### H. Mutation: `HidePost`

> ⏳ GAP petapp-be#961 — `hidePost` chưa có ở backend. Shape below is born-canonical (ready for codegen once the backend ships). Client should not call this mutation until the GAP is resolved.

Hide a specific post from the user's feed.

**Auth:** Required

- `POST` → hide this post only
- `AUTHOR` → hide all posts from this user
- `FAMILY` → hide all posts from this family

**Operation:**
```graphql
mutation HidePost($postId: ID!, $scope: HideScope!) {
  hidePost(postId: $postId, scope: $scope)
}
```

**Variables:**
```json
{ "postId": "post_abc123", "scope": "POST" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "hidePost": true
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `POST_NOT_FOUND` | Post does not exist or has been deleted |

---

### I. Mutation: `UpdatePost`

Edit post (author/family member only).

**Auth:** Required. Returns GraphQL error with code `FORBIDDEN` if caller is not the author or a family member.

**Operation:**
```graphql
mutation UpdatePost($postId: ID!, $input: UpdatePostInput!) {
  updatePost(postId: $postId, input: $input) {
    id
    family {
      id
      name
      avatarUrl
      familyType
    }
    author {
      id
      displayName
      avatarUrl
    }
    pets {
      id
      name
      avatarUrl
    }
    body
    location {
      city
      cityCode
      country
      countryCode
    }
    media {
      order
      sourceType
      embedUrl
      embedProvider
      mediaTag {
        type
        petId
        species
        breed
      }
      media {
        id
        type
        thumbnailUrl
        variants { size key contentType }
        hlsUrl
        blurhash
        signedUrl
        expiresAt
      }
    }
    loveCount
    commentsCount
    isLoved
    visibility
    createdAt
  }
}
```

**Variables:**
```json
{
  "postId": "post_abc123",
  "input": {
    "body": "Updated caption",
    "visibility": "PUBLIC"
  }
}
```

**Response `200 OK`:** Updated post object (same shape as feed post item)
```json
{
  "data": {
    "updatePost": {
      "id": "post_abc123",
      "body": "Updated caption",
      "visibility": "PUBLIC"
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `POST_NOT_FOUND` | Post does not exist or has been deleted |
| `FORBIDDEN` | Caller is not the post author |

---

### J. Mutation: `DeletePost`

Delete post (author/family member only).

**Auth:** Required. Returns GraphQL error with code `FORBIDDEN` if not authorized.

**Operation:**
```graphql
mutation DeletePost($id: ID!) {
  deletePost(id: $id)
}
```

**Variables:**
```json
{ "id": "post_abc123" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "deletePost": true
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `POST_NOT_FOUND` | Post does not exist or has been deleted |
| `FORBIDDEN` | Caller is not the post author |

---

### K. Query: `PostComments`

Fetch comments for the inline comment panel. Returns comments sorted `createdAt asc` (oldest first); client auto-scrolls to bottom on open.

**Auth:** Optional.

**Operation:**
```graphql
query PostComments($postId: ID!, $first: Int = 20, $after: String) {
  post(id: $postId) {
    id
    commentsCount
    comments(first: $first, after: $after) {
      edges {
        cursor
        node {
          id
          author {
            id
            displayName
            avatarUrl
          }
          body
          createdAt
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

**Variables:**
```json
{ "postId": "post_abc123", "first": 10, "after": null }
```

**Response `200 OK`:**
```json
{
  "data": {
    "post": {
      "id": "post_abc123",
      "commentsCount": 34,
      "comments": {
        "edges": [
          {
            "cursor": "eyJpZCI6ImNvbW1lbnRfMDAxIn0=",
            "node": {
              "id": "comment_001",
              "author": {
                "id": "user_002",
                "displayName": "Bella",
                "avatarUrl": "https://cdn.petapp.com/users/user_002/avatar.jpg"
              },
              "body": "Cute quá trời 😍",
              "createdAt": "2026-06-06T07:00:00Z"
            }
          },
          {
            "cursor": "eyJpZCI6ImNvbW1lbnRfMDAyIn0=",
            "node": {
              "id": "comment_002",
              "author": {
                "id": "user_003",
                "displayName": "Quang",
                "avatarUrl": "https://cdn.petapp.com/users/user_003/avatar.jpg"
              },
              "body": "Mắt to ghê 👀",
              "createdAt": "2026-06-06T07:15:00Z"
            }
          }
        ],
        "pageInfo": {
          "hasNextPage": true,
          "endCursor": "eyJpZCI6ImNvbW1lbnRfMDAyIn0="
        }
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `POST_NOT_FOUND` | Post does not exist or has been deleted |

---

### L. Mutation: `CreateComment`

Submit a new comment from the inline panel.

> **Triggers notification:** fires a `NEW_COMMENT` notification to the post author (see screen_22 — Notifications screen). Not fired when the author comments on their own post.

> **Canonical operation:** identical to `CreateComment (N)` in `screen_2_post_detail.md` — same mutation, same return fields. Screen 1 uses it for inline panel top-level comments (`parentId: null`, `replyToUserId: null`).

**Auth:** Required → returns GraphQL error with code `UNAUTHORIZED` if not logged in

**Operation:**
```graphql
mutation CreateComment($postId: ID!, $body: String!, $parentId: ID = null, $replyToUserId: ID = null) {
  createComment(postId: $postId, body: $body, parentId: $parentId, replyToUserId: $replyToUserId) {
    id
    parentId
    author {
      id
      displayName
      avatarUrl
    }
    replyToUser {
      id
      displayName
    }
    body
    createdAt
    replyCount
    isOwn
    isDeletable
  }
}
```

**Variables:**
```json
{ "postId": "post_abc123", "body": "Cute quá trời 😍", "parentId": null, "replyToUserId": null }
```

**Response `200 OK`:**
```json
{
  "data": {
    "createComment": {
      "id": "comment_002",
      "parentId": null,
      "author": {
        "id": "user_001",
        "displayName": "Mochi",
        "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
      },
      "replyToUser": null,
      "body": "Cute quá trời 😍",
      "createdAt": "2026-06-06T08:00:00Z",
      "replyCount": 0,
      "isOwn": true,
      "isDeletable": true
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `POST_NOT_FOUND` | Post does not exist or has been deleted |

---

## User Flow Diagrams

### Feed Load

```
User opens app
  └─> ExploreFeed query (A) { filter: ALL, first: 10 }
        ├─ [unauthenticated] → returns posts with isLoved=false
        └─ [authenticated]   → returns posts with isLoved populated
             └─> Render first 10 posts
                   └─> After post #1, inject Suggested Families widget
                         └─> SuggestedFamilies query (B) { first: 5 }
```

### Infinite Scroll

```
User scrolls to bottom
  └─> ExploreFeed query (A) { filter: ALL, after: <pageInfo.endCursor>, first: 10 }
        └─> Append new posts to list
              └─> pageInfo.hasNextPage=false → show "You're all caught up" state
```

### Suggested Families — Dismiss & Refresh

```
User taps × on a family card
  └─> [authenticated]  → DismissFamilySuggestion mutation (C) { familyId }
  └─> [unauthenticated] → store dismissed id in localStorage/session
  └─> Remove card from widget
        └─> If 0 families remain → widget hides until next reload

User pulls to refresh / navigates away and back
  └─> SuggestedFamilies query (B) { first: 5 }  (server excludes dismissed + already-following)
        └─> New 5 families shown
```

### Follow from Suggested Widget

```
User taps Follow
  └─> [unauthenticated] → redirect to Login
  └─> [authenticated]   → FollowFamily mutation (D) { familyId }
        └─> Button changes to "Following" (or Unfollow on hover/long-press)
```

### Post Context Menu — Hide

```
User taps "..." on a post
  └─> [unauthenticated] → redirect to Login
  └─> [authenticated, not author]
        ├─> "Hide this post"         → HidePost mutation (H) { postId, scope: POST }
        ├─> "Hide posts from @user"  → HidePost mutation (H) { postId, scope: AUTHOR }
        └─> "Hide posts from Family" → HidePost mutation (H) { postId, scope: FAMILY }
              └─> Post (and matching scope) removed from feed immediately
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No posts for selected filter | Show empty state: illustration + message e.g. "No posts yet" |
| `filter=following` and user is not logged in | Show login prompt / redirect to Login |
| Post `pets` list is empty | Use `family.avatarUrl` for post avatar; no subtitle shown |
| Post `pets` list has 1+ items | Still use `family.avatarUrl` for post avatar; subtitle = pet names joined by ` · ` |
| Post has multiple media (>1) | Show carousel with `N/Total` badge in top-right corner; swipeable |
| `mediaTag.type = PET` | Show floating badge bottom-left: look up pet name + avatar in `post.pets` by `mediaTag.petId`; tap → Pet Posts screen |
| `mediaTag.type = RANDOM` | No badge shown on that media frame, regardless of whether `breed` is populated |
| Different media frames link to different pets | Each frame shows its own pet badge independently |
| Post media `sourceType = EMBEDDED` | Show video embed player (YouTube iframe / native player); `media.thumbnailUrl` as poster |
| Suggested widget — 0 families returned | Widget is hidden entirely; not shown at all |
| Family type is `CHARITY` | Show both **Follow** and **Donate** buttons |
| Family type is `NORMAL` | Show **Follow** button only |
| Tapping Donate | Opens a chat with the charity (pre-filled VN support message) — interim, no wallet; see `screen_3` |
| User already follows a suggested family | Should not appear in suggestions (server filters when authenticated) |
| Unauthenticated user on Latest feed | Only `public` posts returned by server |
| Post with `visibility=followers` | Only visible to family members + followers of that family |
| Post with `visibility=private` | Only visible to family members of the authoring family |
| Messages / Notifications button — not logged in | No red dot; tapping redirects to Login |
| Messages red dot source | Unread chats (`UnreadMessageCount BL`); dot only, no number |
| Notifications red dot source | Unread activity (`UnreadNotificationCount BU`); dot only, no number |
| Profile button — not logged in | Default placeholder avatar shown; tapping redirects to Login |
| Love — optimistic update fails (API error) | Revert `loveCount` and `isLoved` to previous state; show error toast |
| Comment count = 0 | Comment button still tappable; inline panel opens showing empty state |
| Comment count ≤ 10 | Show all comments inline (oldest first); no "View all" link needed |
| Comment count > 10 | Show last 10 comments (oldest first within set) + "View all N comments" link → Post Detail screen; auto-scroll to bottom |
| Tapping My Pets / Shops / Services / More while not logged in | Redirect to Login screen |
| Network error on feed load | Show error state with retry button |
| Cursor expired (e.g. after long background) | Reset to first page silently |

---

## Decisions Log

All questions resolved. No open items.

| # | Question | Decision |
|---|----------|----------|
| 1 | Suggested widget empty state | Hide widget entirely when no families remain |
| 2 | Donate button (interim) | No wallet yet — opens a chat with the charity (pre-filled VN message); canonical in `screen_3` |
| 3 | Header unread counts | Fetched separately (not bundled in feed response); **two independent dots** — Messages dot = unread chats (`BL`), Notifications dot = unread activity (`BU`) |
| 4 | Post privacy enforcement | Server-side only — unauthenticated → public only; `followers` → family members + followers; `private` → family members only |
| 5 | `shortDescription` source | Free-text field filled manually by charity family admin in profile settings |
