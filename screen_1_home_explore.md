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
  Right actions: [Search icon] | [Messages icon] | [Profile avatar]

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
| Search | Opens search screen | Opens search screen (no auth needed) |
| Messages | Opens messages screen; shows red dot badge if there are unread messages | Tapping redirects to Login; **no red dot shown** |
| Profile | Shows user's avatar; tapping opens profile screen | Shows default placeholder avatar; tapping redirects to Login |

The Messages unread count is fetched separately (not bundled in the feed response) — via a dedicated endpoint or WebSocket push.

---

### 1. Filter Tabs

3 fixed tabs — no dynamic categories.

| Tab | Description | Auth required |
|-----|-------------|---------------|
| Latest | Newest posts across all families, sorted by `created_at` desc | No |
| Follow | Posts from families the current user follows | Yes — redirect to Login if not authenticated |
| Rescue | Posts from families with `type = charity` | No |

**Inputs:**
- `filter`: `latest` | `following` | `rescue`
- `cursor`: opaque pagination cursor (absent on first load)
- `limit`: default `10`

---

### 2. Post Card

Each post card displays:

| Field | Description |
|-------|-------------|
| `avatar_url` | Display avatar: always use family avatar |
| `family_name` | Name of the family that created the post |
| `author_name` | Display name of the user who created the post |
| `pets` | List of pets linked across all media in this post (can be empty). Used to render subtitle e.g. `"Pudding · Mochi"`. Each item: `{ id, name, avatar_url }` |
| `caption` | Post text / description |
| `media` | List of media items (see Media Object below). Each media item may link to one pet. **Minimum 1 item required** — a post cannot be created without at least one media. |
| `love_count` | Total number of loves |
| `comment_count` | Total number of comments |
| `created_at` | ISO 8601 timestamp |
| `is_loved` | Boolean — whether the current user has loved this post (false if not logged in) |
| `privacy` | `public` \| `followers` \| `private` — see visibility rules below |

**Media Object:**

```json
{
  "id": "string",
  "type": "uploaded" | "embedded",
  "url": "string",
  "thumbnail_url": "string | null",
  "mime_type": "string | null",
  "width": "int | null",
  "height": "int | null",
  "duration_seconds": "float | null",
  "provider": "youtube | vimeo | null",
  "pet": {
    "id": "string",
    "name": "string",
    "avatar_url": "string"
  } | null
}
```

**Rendering rules:**
- `uploaded` → display the file stored on platform (image or video player)
- `embedded` → display embed player (YouTube/Vimeo); use `thumbnail_url` as poster image; `url` is the external video URL
- Multiple media items → swipeable carousel with `N/Total` indicator (e.g. `1/3`) in the top-right corner of the media area

**Pet badge (bottom-left of media):**
- If `media.pet` is non-null → show a floating badge: `[pet avatar] pet name`
- If `media.pet` is null → no badge shown
- Each media item in the carousel may show a different pet badge (or none)
- Tapping the pet badge → navigate to **Pet Posts screen** (shows all posts linked to that pet)
- Different media items in the same post can link to different pets

**Post Card Footer:**

```
287 loves · 34 comments    [Love icon]  Love    [Comment icon]  Comment
```

- **Love button:**
  - Tapping toggles love state (requires login — redirect to Login if not authenticated)
  - **Optimistic update**: `love_count` and `is_loved` are updated immediately in the UI before the API response returns
  - On API error: revert `love_count` and `is_loved` back to previous values and show error toast
  - `287 loves` text is tappable → same behaviour as Love button

- **Comment button / count:**
  - `34 comments` text and Comment button are both tappable
  - Tapping opens an **inline comment panel** (expands below the post card, does not navigate away)
  - Inline panel shows the **latest 10 comments**, sorted by `created_at` desc
  - If `comment_count > 10`: show a "View all N comments" link → navigates to Post Detail screen
  - User can submit a new comment directly from the inline panel (requires login)

- Tapping the **post media or caption area** → opens Post Detail screen (full-screen view with all comments)

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
| `family_id` | Unique ID | all |
| `family_name` | Display name | all |
| `avatar_url` | Family avatar | all |
| `follower_count` | Raw number | all |
| `follower_count_display` | Formatted string, e.g. `"3.6k"` | all |
| `short_description` | Free-text description set by the family | **charity only** — hidden for standard families |
| `family_type` | `standard` \| `charity` | all (drives UI logic) |
| `is_following` | Boolean | all |

**Buttons per family:**
- `standard` type → **Follow** button only
- `charity` type → **Follow** button + **Donate** button

**Donate button behaviour:**
- Tapping Donate → opens in-app Donate screen for that family
- Current state: Donate screen shows a "Coming Soon" placeholder page
- No external URL needed; navigation is handled in-app by family ID

**Dismiss / Hide:**
- Each family row has an `×` (dismiss) button
- Tapping `×` marks that family as "dismissed" for this session
- Dismissed families are excluded from future suggestions **within this session**
- On next page reload / app restart, a new set of 5 suggestions is shown (excluding any permanently dismissed ones if the user is logged in)
- If user is logged in: dismissal is persisted server-side (never suggest this family again until user clears history)
- If user is not logged in: dismissal is session-only (localStorage / in-memory)

**Inputs (API):**
- `limit`: `5` (fixed)
- `exclude_ids`: list of family IDs to exclude (previously dismissed)
- `seed` or `session_id`: for randomisation so that refresh returns a new set

---

## API Endpoints Required

### A. `GET /feed/explore`

Fetches the paginated post feed for the Explore screen.

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `filter` | string | No | `latest` | `latest` \| `following` \| `rescue` |
| `cursor` | string | No | — | Pagination cursor from previous response |
| `limit` | int | No | `10` | Number of posts per page |

**Headers:**
- `Authorization: Bearer <token>` — optional. When present, `is_loved` and viewer-specific fields are populated.

**Response `200 OK`:**

```json
{
  "posts": [
    {
      "id": "post_abc123",
      "family": {
        "id": "fam_xyz",
        "name": "Pudding's Family",
        "avatar_url": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
        "type": "standard"
      },
      "author": {
        "id": "user_001",
        "display_name": "Mochi",
        "avatar_url": "https://cdn.petapp.com/users/user_001/avatar.jpg"
      },
      "pets": [
        {
          "id": "pet_111",
          "name": "Pudding",
          "avatar_url": "https://cdn.petapp.com/pets/pet_111/avatar.jpg"
        },
        {
          "id": "pet_222",
          "name": "Mochi",
          "avatar_url": "https://cdn.petapp.com/pets/pet_222/avatar.jpg"
        }
      ],
      "caption": "Pudding nằm chờ mama nấu cơm 🌕 Q7 cat life",
      "media": [
        {
          "id": "media_001",
          "type": "uploaded",
          "url": "https://cdn.petapp.com/media/001.jpg",
          "thumbnail_url": null,
          "mime_type": "image/jpeg",
          "width": 1080,
          "height": 1080,
          "duration_seconds": null,
          "provider": null,
          "pet": {
            "id": "pet_111",
            "name": "Pudding",
            "avatar_url": "https://cdn.petapp.com/pets/pet_111/avatar.jpg"
          }
        },
        {
          "id": "media_002",
          "type": "embedded",
          "url": "https://www.youtube.com/watch?v=abc123",
          "thumbnail_url": "https://img.youtube.com/vi/abc123/hqdefault.jpg",
          "mime_type": null,
          "width": null,
          "height": null,
          "duration_seconds": 62.0,
          "provider": "youtube",
          "pet": {
            "id": "pet_222",
            "name": "Mochi",
            "avatar_url": "https://cdn.petapp.com/pets/pet_222/avatar.jpg"
          }
        },
        {
          "id": "media_003",
          "type": "uploaded",
          "url": "https://cdn.petapp.com/media/003.jpg",
          "thumbnail_url": null,
          "mime_type": "image/jpeg",
          "width": 1080,
          "height": 1080,
          "duration_seconds": null,
          "provider": null,
          "pet": null
        }
      ],
      "love_count": 287,
      "comment_count": 34,
      "is_loved": false,
      "privacy": "public",
      "created_at": "2026-06-06T06:00:00Z"
    }
  ],
  "next_cursor": "eyJpZCI6InBvc3RfYWJjMTIzIn0=",
  "has_more": true
}
```

> Example above: post has 3 media. Media 1 (uploaded image) → linked to Pudding. Media 2 (embedded YouTube) → linked to Mochi. Media 3 (uploaded image) → no pet linked, no badge shown.

**Notes:**
- `media` list must have at least 1 item — enforced at post creation; the feed API will never return a post with empty `media`
- `pets` is the deduplicated list of pets linked across all media items in the post; can be empty `[]`
- Post header avatar: always use `family.avatar_url`
- Post header subtitle: render pet names from `pets` list joined by ` · ` (e.g. `"Pudding · Mochi"`); omit if `pets` is empty
- `filter=following` requires authentication → return `401` if no valid token
- `filter=rescue` returns posts from families where `family.type = charity`
- `is_loved` is always `false` when unauthenticated
- Server enforces privacy rules before returning results — unauthenticated callers only receive `public` posts; `followers` posts are filtered based on the caller's follow list

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_FILTER` | Unknown filter value |
| `401` | `UNAUTHORIZED` | `filter=following` without auth token |
| `422` | `INVALID_CURSOR` | Cursor is malformed or expired |

---

### B. `GET /families/suggested`

Returns families to show in the Suggested Families widget.

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `limit` | int | No | `5` | How many families to return |
| `exclude_ids` | string (comma-separated) | No | — | Family IDs to exclude |
| `seed` | string | No | — | Random seed for deterministic shuffle per session |

**Headers:**
- `Authorization: Bearer <token>` — optional. When present, excludes already-followed families and server-side dismissed families.

**Response `200 OK`:**

```json
{
  "families": [
    {
      "id": "fam_cat_house",
      "name": "My's Cat House",
      "avatar_url": "https://cdn.petapp.com/families/fam_cat_house/avatar.jpg",
      "follower_count": 3600,
      "follower_count_display": "3.6k",
      "short_description": "Rescue & rehome cats in HCM City",
      "type": "charity",
      "is_following": false
    },
    {
      "id": "fam_normal_001",
      "name": "Mochi's Family",
      "avatar_url": "https://cdn.petapp.com/families/fam_normal_001/avatar.jpg",
      "follower_count": 420,
      "follower_count_display": "420",
      "short_description": null,
      "type": "standard",
      "is_following": false
    }
  ]
}
```

**Note:** `short_description` is always present in the response but is `null` for `standard` families. The client should only render the description line when `type = charity` and `short_description` is non-null.

`short_description` is a free-text field that the charity family admin fills in manually (via their family profile settings). It is not computed or auto-generated.

---

### C. `POST /families/{family_id}/dismiss-suggestion`

Marks a family as dismissed from suggestions for the authenticated user.

**Auth:** Required (for unauthenticated users, dismissal is handled client-side only)

**Response `204 No Content`**

---

### D. `POST /families/{family_id}/follow`

Follow a family.

**Auth:** Required → `401` if not logged in

**Response `200 OK`:**
```json
{ "is_following": true, "follower_count": 3601 }
```

---

### E. `DELETE /families/{family_id}/follow`

Unfollow a family.

**Auth:** Required

**Response `200 OK`:**
```json
{ "is_following": false, "follower_count": 3600 }
```

---

### F. `POST /posts/{post_id}/love`

Love a post.

**Auth:** Required → redirect to login if not authenticated

**Response `200 OK`:**
```json
{ "is_loved": true, "love_count": 288 }
```

---

### G. `DELETE /posts/{post_id}/love`

Un-love a post.

**Auth:** Required

**Response `200 OK`:**
```json
{ "is_loved": false, "love_count": 287 }
```

---

### H. `POST /posts/{post_id}/hide`

Hide a specific post from the user's feed.

**Auth:** Required

**Body:**
```json
{ "scope": "post" | "author" | "family" }
```

- `post` → hide this post only
- `author` → hide all posts from this user
- `family` → hide all posts from this family

**Response `204 No Content`**

---

### I. `PATCH /posts/{post_id}`

Edit post (author/family member only).

**Auth:** Required. Returns `403` if caller is not the author or a family member.

**Body (partial update):**
```json
{
  "caption": "Updated caption",
  "privacy": "public" | "followers" | "private"
}
```

**Response `200 OK`:** Updated post object (same shape as feed post item)

---

### J. `DELETE /posts/{post_id}`

Delete post (author/family member only).

**Auth:** Required. Returns `403` if not authorized.

**Response `204 No Content`**

---

### K. `GET /posts/{post_id}/comments`

Fetch comments for the inline comment panel.

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `limit` | int | No | `10` | Number of comments to return |
| `cursor` | string | No | — | Pagination cursor (for "View all" flow in Post Detail) |

**Response `200 OK`:**

```json
{
  "comments": [
    {
      "id": "comment_001",
      "author": {
        "id": "user_002",
        "display_name": "Bella",
        "avatar_url": "https://cdn.petapp.com/users/user_002/avatar.jpg"
      },
      "body": "Cute quá trời 😍",
      "created_at": "2026-06-06T07:00:00Z"
    }
  ],
  "total_count": 34,
  "next_cursor": "eyJpZCI6ImNvbW1lbnRfMDAxIn0=",
  "has_more": true
}
```

---

### L. `POST /posts/{post_id}/comments`

Submit a new comment from the inline panel.

**Auth:** Required → `401` if not logged in

**Body:**
```json
{ "body": "Cute quá trời 😍" }
```

**Response `201 Created`:**
```json
{
  "id": "comment_002",
  "author": {
    "id": "user_001",
    "display_name": "Mochi",
    "avatar_url": "https://cdn.petapp.com/users/user_001/avatar.jpg"
  },
  "body": "Cute quá trời 😍",
  "created_at": "2026-06-06T08:00:00Z"
}
```

---

## User Flow Diagrams

### Feed Load

```
User opens app
  └─> GET /feed/explore?filter=latest&limit=10
        ├─ [unauthenticated] → returns posts with is_loved=false
        └─ [authenticated]   → returns posts with is_loved populated
             └─> Render first 10 posts
                   └─> After post #1, inject Suggested Families widget
                         └─> GET /families/suggested?limit=5
```

### Infinite Scroll

```
User scrolls to bottom
  └─> GET /feed/explore?filter=latest&cursor=<next_cursor>&limit=10
        └─> Append new posts to list
              └─> has_more=false → show "You're all caught up" state
```

### Suggested Families — Dismiss & Refresh

```
User taps × on a family card
  └─> [authenticated]  → POST /families/{id}/dismiss-suggestion
  └─> [unauthenticated] → store dismissed id in localStorage/session
  └─> Remove card from widget
        └─> If 0 families remain → widget hides until next reload

User pulls to refresh / navigates away and back
  └─> GET /families/suggested?limit=5&exclude_ids=<dismissed_ids>
        └─> New 5 families shown (server excludes dismissed + already-following)
```

### Follow from Suggested Widget

```
User taps Follow
  └─> [unauthenticated] → redirect to Login
  └─> [authenticated]   → POST /families/{id}/follow
        └─> Button changes to "Following" (or Unfollow on hover/long-press)
```

### Post Context Menu — Hide

```
User taps "..." on a post
  └─> [unauthenticated] → redirect to Login
  └─> [authenticated, not author]
        ├─> "Hide this post"         → POST /posts/{id}/hide  { scope: "post" }
        ├─> "Hide posts from @user"  → POST /posts/{id}/hide  { scope: "author" }
        └─> "Hide posts from Family" → POST /posts/{id}/hide  { scope: "family" }
              └─> Post (and matching scope) removed from feed immediately
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No posts for selected filter | Show empty state: illustration + message e.g. "No posts yet" |
| `filter=following` and user is not logged in | Show login prompt / redirect to Login |
| Post `pets` list is empty | Use `family.avatar_url` for post avatar; no subtitle shown |
| Post `pets` list has 1+ items | Still use `family.avatar_url` for post avatar; subtitle = pet names joined by ` · ` |
| Post has multiple media (>1) | Show carousel with `N/Total` badge in top-right corner; swipeable |
| Media has `pet` linked | Show floating badge bottom-left: pet avatar + pet name; tap → Pet Posts screen |
| Media has no `pet` linked (`pet: null`) | No badge shown on that media frame |
| Different media frames link to different pets | Each frame shows its own pet badge independently |
| Post media type is `embedded` | Show video embed player (YouTube iframe / native player); `thumbnail_url` as poster |
| Suggested widget — 0 families returned | Widget is hidden entirely; not shown at all |
| Family type is `charity` | Show both **Follow** and **Donate** buttons |
| Family type is `standard` | Show **Follow** button only |
| Tapping Donate | Navigate to in-app Donate screen (currently "Coming Soon" page) |
| User already follows a suggested family | Should not appear in suggestions (server filters when authenticated) |
| Unauthenticated user on Latest feed | Only `public` posts returned by server |
| Post with `privacy=followers` | Only visible to family members + followers of that family |
| Post with `privacy=private` | Only visible to family members of the authoring family |
| Messages button — not logged in | No red dot; tapping redirects to Login |
| Profile button — not logged in | Default placeholder avatar shown; tapping redirects to Login |
| Love — optimistic update fails (API error) | Revert `love_count` and `is_loved` to previous state; show error toast |
| Comment count = 0 | Comment button still tappable; inline panel opens showing empty state |
| Comment count ≤ 10 | Show all comments inline; no "View all" link needed |
| Comment count > 10 | Show latest 10 inline + "View all N comments" link → Post Detail screen |
| Tapping My Pets / Shops / Services / More while not logged in | Redirect to Login screen |
| Network error on feed load | Show error state with retry button |
| Cursor expired (e.g. after long background) | Reset to first page silently |

---

## Decisions Log

All questions resolved. No open items.

| # | Question | Decision |
|---|----------|----------|
| 1 | Suggested widget empty state | Hide widget entirely when no families remain |
| 2 | Donate button destination | In-app screen; currently shows "Coming Soon" page |
| 3 | Messages unread count | Fetched separately (not bundled in feed response) |
| 4 | Post privacy enforcement | Server-side only — unauthenticated → public only; `followers` → family members + followers; `private` → family members only |
| 5 | `short_description` source | Free-text field filled manually by charity family admin in profile settings |
