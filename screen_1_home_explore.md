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
| Messages | Opens the **Notifications screen** (screen_10); shows a **red dot** when there is any unread — combined across Chats + general Notifications | Tapping redirects to Login; **no red dot shown** |
| Profile | Shows user's avatar; tapping opens profile screen | Shows default placeholder avatar; tapping redirects to Login |

The unread state is fetched separately (not bundled in the feed response) — combining `UnreadMessageCount (BL)` + `NotificationUnreadCount (BU)` (see screen_10); delivered via WebSocket push, falling back to polling. The dot shows whenever the combined count `> 0` (no number).

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

> **Canonical definition.** This post card layout applies to ALL screens that display posts: Explore feed, Post Detail, Family Posts, Pet Posts, Random Pet Posts, User Posts, and Loved Posts. Any update here propagates to all screens.

Each post card displays:

| Field | Description |
|-------|-------------|
| `avatar_url` | Display avatar: always use family avatar |
| `family_name` | Name of the family that created the post |
| `author_name` | Display name of the user who created the post. Shown top-right: `"author_name · time"` |
| `pets` | List of **named pets** linked across all media in this post (only media with `media_tag.type = "pet"`). Used to render subtitle e.g. `"Pudding · Mochi"`. Each item: `{ id, name, avatar_url }`. Can be empty. |
| `caption` | Post text / description |
| `location` | Optional. `{ "city": "Hồ Chí Minh", "city_code": "HCM", "country": "Việt Nam", "country_code": "VN" }`. Shown bottom-right below author/time line: `"HCM - VN"`. Omitted if `null`. |
| `media` | List of media items (see Media Object below). **Minimum 1, maximum 10.** |
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
  "media_tag": {
    "type": "pet" | "random",
    "id": "string | null",
    "species": "string | null",
    "breed": "string | null"
  }
}
```

**`media_tag` types:**
| Type | Meaning | `id` | `species` | `breed` |
|------|---------|------|-----------|---------|
| `pet` | AI matched to a named pet in the family | `pet_id` — matches an entry in `post.pets` | species of the matched pet | breed of the matched pet |
| `random` | AI detected no named pet match, or media is `embedded` | `null` | species string if AI detected an animal (e.g. `"cat"`), or `null` | breed string if AI identified a specific breed, or `null` |

**Rendering rules:**
- `uploaded` → display the file stored on platform (image or video player)
- `embedded` → display embed player (YouTube/Vimeo); use `thumbnail_url` as poster image; `url` is the external video URL
- Multiple media items → swipeable carousel with `N/Total` indicator (e.g. `1/3`) in the top-right corner of the media area

**Pet badge (bottom-left of media):**
- `media_tag.type = "pet"` → show floating badge: `[pet avatar]  pet name`; look up display info using `media_tag.id` against `post.pets` (where `pets[].id = media_tag.id`); tappable → Pet Posts screen
- `media_tag.type = "random"` → **no badge shown** (regardless of whether `breed`/`species` is populated; breed/species shown in Random Pets section context only)
- Each media item in the carousel evaluates independently

**Private pet enforcement (server-side):**
- If `pet.is_public = false` AND viewer is not a family member → server returns `media_tag` as `{ type: "random", species: ..., breed: ... }` instead of `{ type: "pet", id: ..., ... }`
- Client applies existing rules: `type = "random"` → no badge; no special client logic needed

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
  - Inline panel shows the **last 10 comments**, sorted by `created_at` asc (oldest first within the set); panel **auto-scrolls to bottom** on open so the newest comment is visible
  - If `comment_count > 10`: show a "View all N comments" link → navigates to Post Detail screen
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

All API calls go to a single `POST /graphql` endpoint.
**Auth header:** `Authorization: Bearer <token>` (optional unless stated otherwise).

---

### A. Query: `Feed`

Fetches the paginated post feed for the Explore screen.

**Auth:** Optional. When present, `isLoved` and viewer-specific fields are populated.

**Operation:**
```graphql
query Feed($filter: FeedFilter, $cursor: String, $limit: Int) {
  feed(filter: $filter, cursor: $cursor, limit: $limit) {
    posts {
      id
      family {
        id
        name
        avatarUrl
        type
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
      caption
      location {
        city
        cityCode
        country
        countryCode
      }
      media {
        id
        type
        url
        thumbnailUrl
        mimeType
        width
        height
        durationSeconds
        provider
        mediaTag {
          type
          id
          species
          breed
        }
      }
      loveCount
      commentCount
      isLoved
      privacy
      createdAt
    }
    nextCursor
    hasMore
  }
}
```

**Variables:**
```json
{ "filter": "LATEST", "cursor": null, "limit": 10 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "feed": {
      "posts": [
        {
          "id": "post_abc123",
          "family": {
            "id": "fam_xyz",
            "name": "Pudding's Family",
            "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
            "type": "STANDARD"
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
          "caption": "Pudding nằm chờ mama nấu cơm 🌕 Q7 cat life",
          "location": {
            "city": "Hồ Chí Minh",
            "cityCode": "HCM",
            "country": "Việt Nam",
            "countryCode": "VN"
          },
          "media": [
            {
              "id": "media_001",
              "type": "UPLOADED",
              "url": "https://cdn.petapp.com/media/001.jpg",
              "thumbnailUrl": null,
              "mimeType": "image/jpeg",
              "width": 1080,
              "height": 1080,
              "durationSeconds": null,
              "provider": null,
              "mediaTag": {
                "type": "PET",
                "id": "pet_111",
                "species": "Cat",
                "breed": "Orange Tabby Cat"
              }
            },
            {
              "id": "media_002",
              "type": "EMBEDDED",
              "url": "https://www.youtube.com/watch?v=abc123",
              "thumbnailUrl": "https://img.youtube.com/vi/abc123/hqdefault.jpg",
              "mimeType": null,
              "width": null,
              "height": null,
              "durationSeconds": 62.0,
              "provider": "YOUTUBE",
              "mediaTag": {
                "type": "RANDOM",
                "id": null,
                "species": null,
                "breed": null
              }
            },
            {
              "id": "media_003",
              "type": "UPLOADED",
              "url": "https://cdn.petapp.com/media/003.jpg",
              "thumbnailUrl": null,
              "mimeType": "image/jpeg",
              "width": 1080,
              "height": 1080,
              "durationSeconds": null,
              "provider": null,
              "mediaTag": {
                "type": "RANDOM",
                "id": null,
                "breed": "British Shorthair"
              }
            }
          ],
          "loveCount": 287,
          "commentCount": 34,
          "isLoved": false,
          "privacy": "PUBLIC",
          "createdAt": "2026-06-06T06:00:00Z"
        }
      ],
      "nextCursor": "eyJpZCI6InBvc3RfYWJjMTIzIn0=",
      "hasMore": true
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

**Time display rules (based on `created_at`):**

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
- `filter: FOLLOWING` requires authentication → returns GraphQL error with code `UNAUTHORIZED` if no valid token
- `filter: RESCUE` returns posts from families where `family.type = CHARITY`
- `isLoved` is always `false` when unauthenticated
- Server enforces privacy rules before returning results — unauthenticated callers only receive `PUBLIC` posts; `FOLLOWERS` posts are filtered based on the caller's follow list

**Errors:**

| Code | Scenario |
|------|----------|
| `INVALID_FILTER` | Unknown filter value |
| `UNAUTHORIZED` | `filter: FOLLOWING` without auth token |
| `INVALID_CURSOR` | Cursor is malformed or expired |

---

### B. Query: `SuggestedFamilies`

Returns families to show in the Suggested Families widget.

**Auth:** Optional. When present, excludes already-followed families and server-side dismissed families.

**Operation:**
```graphql
query SuggestedFamilies($limit: Int, $excludeIds: [ID!], $seed: String) {
  suggestedFamilies(limit: $limit, excludeIds: $excludeIds, seed: $seed) {
    id
    name
    avatarUrl
    followerCount
    followerCountDisplay
    shortDescription
    type
    isFollowing
  }
}
```

**Variables:**
```json
{ "limit": 5, "excludeIds": [], "seed": "session_abc" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "suggestedFamilies": [
      {
        "id": "fam_cat_house",
        "name": "My's Cat House",
        "avatarUrl": "https://cdn.petapp.com/families/fam_cat_house/avatar.jpg",
        "followerCount": 3600,
        "followerCountDisplay": "3.6k",
        "shortDescription": "Rescue & rehome cats in HCM City",
        "type": "CHARITY",
        "isFollowing": false
      },
      {
        "id": "fam_normal_001",
        "name": "Mochi's Family",
        "avatarUrl": "https://cdn.petapp.com/families/fam_normal_001/avatar.jpg",
        "followerCount": 420,
        "followerCountDisplay": "420",
        "shortDescription": null,
        "type": "STANDARD",
        "isFollowing": false
      }
    ]
  }
}
```

**Note:** `shortDescription` is always present in the response but is `null` for `STANDARD` families. The client should only render the description line when `type = CHARITY` and `shortDescription` is non-null.

`shortDescription` is a free-text field that the charity family admin fills in manually (via their family profile settings). It is not computed or auto-generated.

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

> **Triggers notification:** fires a `NEW_FOLLOWER` notification to the followed family's members (see screen_10 → Notifications tab). Unfollowing (E) does not notify.

**Auth:** Required → returns GraphQL error with code `UNAUTHORIZED` if not logged in

**Operation:**
```graphql
mutation FollowFamily($familyId: ID!) {
  followFamily(familyId: $familyId) {
    isFollowing
    followerCount
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
      "isFollowing": true,
      "followerCount": 3601
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
    isFollowing
    followerCount
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
      "isFollowing": false,
      "followerCount": 3600
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

> **Triggers notification:** fires a `POST_LOVES` notification to the post author — **grouped** per post (`{actor} and {N} others loved your post`); see screen_10 → Notifications tab. Un-loving (G) does not notify.

**Auth:** Required → redirect to login if not authenticated

**Operation:**
```graphql
mutation LovePost($postId: ID!) {
  lovePost(postId: $postId) {
    isLoved
    loveCount
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
      "isLoved": true,
      "loveCount": 288
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
    isLoved
    loveCount
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
      "isLoved": false,
      "loveCount": 287
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
      type
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
    caption
    location {
      city
      cityCode
      country
      countryCode
    }
    media {
      id
      type
      url
      thumbnailUrl
      mimeType
      width
      height
      durationSeconds
      provider
      mediaTag {
        type
        id
        species
        breed
      }
    }
    loveCount
    commentCount
    isLoved
    privacy
    createdAt
  }
}
```

**Variables:**
```json
{
  "postId": "post_abc123",
  "input": {
    "caption": "Updated caption",
    "privacy": "PUBLIC"
  }
}
```

**Response `200 OK`:** Updated post object (same shape as feed post item)
```json
{
  "data": {
    "updatePost": {
      "id": "post_abc123",
      "caption": "Updated caption",
      "privacy": "PUBLIC"
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
mutation DeletePost($postId: ID!) {
  deletePost(postId: $postId)
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
query PostComments($postId: ID!, $limit: Int, $cursor: String) {
  postComments(postId: $postId, limit: $limit, cursor: $cursor) {
    comments {
      id
      author {
        id
        displayName
        avatarUrl
      }
      body
      createdAt
    }
    totalCount
    nextCursor
    hasMore
  }
}
```

**Variables:**
```json
{ "postId": "post_abc123", "limit": 10, "cursor": null }
```

**Response `200 OK`:**
```json
{
  "data": {
    "postComments": {
      "comments": [
        {
          "id": "comment_001",
          "author": {
            "id": "user_002",
            "displayName": "Bella",
            "avatarUrl": "https://cdn.petapp.com/users/user_002/avatar.jpg"
          },
          "body": "Cute quá trời 😍",
          "createdAt": "2026-06-06T07:00:00Z"
        },
        {
          "id": "comment_002",
          "author": {
            "id": "user_003",
            "displayName": "Quang",
            "avatarUrl": "https://cdn.petapp.com/users/user_003/avatar.jpg"
          },
          "body": "Mắt to ghê 👀",
          "createdAt": "2026-06-06T07:15:00Z"
        }
      ],
      "totalCount": 34,
      "nextCursor": "eyJpZCI6ImNvbW1lbnRfMDAxIn0=",
      "hasMore": true
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

> **Triggers notification:** fires a `NEW_COMMENT` notification to the post author (see screen_10 → Notifications tab). Not fired when the author comments on their own post.

**Auth:** Required → returns GraphQL error with code `UNAUTHORIZED` if not logged in

**Operation:**
```graphql
mutation CreateComment($postId: ID!, $input: CreateCommentInput!) {
  createComment(postId: $postId, input: $input) {
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
```

**Variables:**
```json
{ "postId": "post_abc123", "input": { "body": "Cute quá trời 😍" } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "createComment": {
      "id": "comment_002",
      "author": {
        "id": "user_001",
        "displayName": "Mochi",
        "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
      },
      "body": "Cute quá trời 😍",
      "createdAt": "2026-06-06T08:00:00Z"
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
  └─> Feed query (A) { filter: LATEST, limit: 10 }
        ├─ [unauthenticated] → returns posts with isLoved=false
        └─ [authenticated]   → returns posts with isLoved populated
             └─> Render first 10 posts
                   └─> After post #1, inject Suggested Families widget
                         └─> SuggestedFamilies query (B) { limit: 5 }
```

### Infinite Scroll

```
User scrolls to bottom
  └─> Feed query (A) { filter: LATEST, cursor: <next_cursor>, limit: 10 }
        └─> Append new posts to list
              └─> hasMore=false → show "You're all caught up" state
```

### Suggested Families — Dismiss & Refresh

```
User taps × on a family card
  └─> [authenticated]  → DismissFamilySuggestion mutation (C) { familyId }
  └─> [unauthenticated] → store dismissed id in localStorage/session
  └─> Remove card from widget
        └─> If 0 families remain → widget hides until next reload

User pulls to refresh / navigates away and back
  └─> SuggestedFamilies query (B) { limit: 5 }  (server excludes dismissed + already-following)
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
| Post `pets` list is empty | Use `family.avatar_url` for post avatar; no subtitle shown |
| Post `pets` list has 1+ items | Still use `family.avatar_url` for post avatar; subtitle = pet names joined by ` · ` |
| Post has multiple media (>1) | Show carousel with `N/Total` badge in top-right corner; swipeable |
| `media_tag.type = "pet"` | Show floating badge bottom-left: look up pet name + avatar in `post.pets` by `media_tag.id`; tap → Pet Posts screen |
| `media_tag.type = "random"` | No badge shown on that media frame, regardless of whether `breed` is populated |
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
| Messages red dot source | Combined unread of Chats + general Notifications (`BL` + `BU`); dot only, no number |
| Profile button — not logged in | Default placeholder avatar shown; tapping redirects to Login |
| Love — optimistic update fails (API error) | Revert `love_count` and `is_loved` to previous state; show error toast |
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
| 2 | Donate button destination | In-app screen; currently shows "Coming Soon" page |
| 3 | Messages unread count | Fetched separately (not bundled in feed response); red dot = combined Chats + general Notifications unread (`BL` + `BU`) |
| 4 | Post privacy enforcement | Server-side only — unauthenticated → public only; `followers` → family members + followers; `private` → family members only |
| 5 | `short_description` source | Free-text field filled manually by charity family admin in profile settings |
