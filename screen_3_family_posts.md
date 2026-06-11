# Screen 3: Family Posts

## Overview

Profile and post feed for a single family — shown to **unauthenticated users and non-members only**.  
When a logged-in family member taps their own family name → redirected to **My Pets screen** (screen_8) instead.  
Navigated to from: tapping a family name on any post card (Explore feed, Post Detail, etc.) when the viewer is not a member of that family.  
Accessible without login — unauthenticated users can view everything. Actions (Follow, Message) require login.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: Family name (static)

[Scrollable area]
  ├── Family Info Card
  │     ├── Avatar (stacked pet photos or default) + "CHARITY" ribbon badge if type=charity
  │     ├── Family name
  │     ├── @tag  ·  City, Country
  │     ├── N pets  [· N randoms — only if > 0]  · N followers
  │     └── [Follow button]  [Message button]
  │
  ├── Charity Section  ← only shown when familyType = CHARITY
  │     ├── [♡ icon]  shortDescription
  │     ├── [♡ icon]  "N people have helped"
  │     └── [Donate button — full width]
  │
  ├── Pets List  ← horizontal scroll when petCount > 5
  │     Column 1 (pets 1–5)    Column 2 (pets 6–10)    ...
  │     ├── Pet row 1           ├── Pet row 6
  │     ├── Pet row 2           ├── Pet row 7
  │     ├── Pet row 3           ├── Pet row 8
  │     ├── Pet row 4           ├── Pet row 9
  │     └── Pet row 5           └── Pet row 10
  │     [Random Pets row — only if randomCount > 0, always in last column]
  │
  ├── Parents row
  │     └── "Parents"  [count badge]  [>]  → opens bottom sheet popup
  │
  ├── About section
  │     └── Description text
  │
  └── Posts section
        ├── "POSTS"  [Grid view icon]  [List view icon]
        └── Post list (list view) or media grid (grid view)
```

---

## Components

### 1. Family Info Card

| Field | Description |
|-------|-------------|
| `name` | Display name of the family |
| `tag` | Unique identifier, prefixed with `@` (e.g. `@minhfamily`) |
| `city` | City name |
| `country` | Country code or name (e.g. `VN`) |
| `avatarUrl` | Used as fallback / single avatar |
| `petAvatars` | Ordered list of pet avatar URLs for the stacked display (up to 5) |
| `petCount` | Number of actual pets in the family |
| `randomCount` | Number of media items where `mediaTag.type = "random" AND (breed IS NOT NULL OR species IS NOT NULL)` — AI detected a breed or species but could not match to a named pet. Does NOT count media with no breed or species detected. Hidden from stats line when `0`. |
| `randomPostCount` | Total posts linked to random pets in this family. Used in the Random Pets row. |
| `social.followersCount` | Total followers |
| `familyType` | `STANDARD` \| `CHARITY` |
| `social.isFollowedByMe` | Boolean (false when unauthenticated) |
| `bio` | Free-text description |
| `shortDescription` | Short tagline for charity families (e.g. `"Home-based cat rescue"`). `null` for standard families. |
| `donorCount` | Number of people who have donated. Only meaningful when `familyType = CHARITY`; `null` for standard families. |

> followersCount trả số thô; client tự format "3.6k" theo locale (bỏ followerCountDisplay — i18n client-side).

> PR #882: `viewerRole` (FamilyRole) và `isPrimary` (Boolean) nay khả dụng trên family object — có thể dùng cho UI logic quyền hạn thành viên nếu cần.

**Avatar display rules:**
- Family avatar can be explicitly set by the owner (via Screen 6 Update Family); if not set, falls back to stacked pet avatars
- `petCount = 0` (public pets) → show single default family avatar
- `petCount = 1` (public pet) → show that pet's avatar
- `petCount ≥ 2` (public pets) → show stacked overlapping pet avatars (up to 5); auto-rotates through them on a timer (client-side animation only, no API call)
- Only **public pets** (`isPublic = true`) are included in the stacked display — private pets are never shown to non-members
- `familyType = CHARITY` → show a **"CHARITY" ribbon badge** overlaid on the avatar (top-left corner)

**Follow button:**
- Not following → button label "Follow"; tap → `FollowFamily mutation (D)` (requires login → redirect to Login)
- Following → button label "Following"; tap → `UnfollowFamily mutation (E)` → button reverts to "Follow" immediately (optimistic); show undo toast for **5 seconds**: *"Unfollowed [Family Name]"* + **[Undo]** button → if Undo tapped: re-follow silently (no second toast), button returns to "Following"
- Reuse endpoints D and E from Screen 1

**Message button:** *(hidden when you are a member of this family — active or non-active; you're already on the receiving side, can't message your own family; see screen_10)*
- Tap → show **"Send as…"** bottom sheet (your individual self **or** your active family), then `StartThread (BI)` → open the thread in the Messages screen's Thread View (screen_10). If a thread already exists for the same sender ↔ this family, it is reused (no duplicate).
- Requires login → redirect to Login if not authenticated.

---

### 2. Charity Section *(only rendered when `familyType = CHARITY`)*

Displayed between the Family Info Card and the Pets List.

| Element | Description |
|---------|-------------|
| `shortDescription` | Displayed with a ♡ icon prefix, e.g. `"♡ Home-based cat rescue"` |
| `donorCount` | Displayed as `"♡ 184 people have helped"` |
| Donate button | Full-width button — see **Donate button behaviour** below |

- The entire section is hidden for `STANDARD` families.

**Donate button behaviour (canonical — interim, no wallet yet):**

Tapping **Donate** does **not** open a payment/wallet screen this phase. Instead it **opens a chat with the charity family**, pre-filled with a Vietnamese support message — the same messaging mechanic as Lost Pet **"I saw"** (`screen_19`) and Rescue **"Inquire to Adopt"** (`screen_26`).

- Not logged in → **redirect to Login** (return here after).
- Logged in → **"Send as…"** (self / active family) → `StartThread (BI)` `{ receiverType: FAMILY, receiverId: family.id, senderType: USER | FAMILY }` → open the thread in **Messages** (`screen_10`), **pre-filled** with:
  > *"Chào {familyName}, mình muốn ủng hộ cho hoạt động cứu trợ của nhà mình ạ 🐾 Cho mình hỏi có thể đóng góp bằng cách nào được không?"*
- Reuses an existing thread if one exists (no duplicate).
- **Hidden** for members of this charity family (can't message your own family — `screen_10` rule).
- `donorCount` is **not** updated by this interim flow (no real donation happens yet); it stays a future field, to be driven by the wallet/donation flow when that ships (see Open Items).

> This canonical behaviour is referenced by every other Donate button — Explore (`screen_1`), Profile Settings (`screen_5`), Search (`screen_11`), Following (`screen_16`), and Rescue Detail (`screen_26`).

---

### 3. Pets List

**Only public pets shown** (`isPublic = true`). Private pets are hidden from this screen entirely — Family Posts is viewed by non-members only; members see all pets (including private) in My Pets (screen_8).

**Layout rules:**
- `petCount ≤ 5` → single vertical column (current behaviour)
- `petCount > 5` → **horizontal scroll**, 5 rows per column; swipe left to reveal more columns
  - Column 1: pets 1–5, Column 2: pets 6–10, etc.
  - Next column's avatars peek slightly on the right edge to indicate scrollability
  - Random Pets row (if applicable) always appears as the last item in the final column

Each pet row:

| Field | Description |
|-------|-------------|
| `id` | Pet ID |
| `avatarUrl` | Pet avatar |
| `name` | Pet name |
| `breed` | `BreedGQL` — use `breed.nameVi` (Vietnamese) or `breed.nameEn` for display (may be truncated with `...` if long); `null` if unknown |
| `sex` | `male` \| `female` \| `unknown` |
| `ageMonths` | Tổng số tháng tuổi (Int). Client render "3 tuổi"/"3 years" theo locale + birthDatePrecision. |
| `postCount` | Total posts linked to this pet |

> Tuổi trả về dạng `ageMonths` (Int) — client tự format hiển thị theo locale + `birthDatePrecision`, server không format chuỗi.

**Interactions:**
- Tap anywhere on the row → navigates to **Pet Posts screen** for that pet (viewer is always unauth/non-member on this screen)
- **No Story button** → the pet **Story** (`screen_29`) is **member-only**, and this screen's viewer is always a non-member, so it is **not shown here** (members reach Story from My Pets / Pet Detail)

**Random Pets row** (shown only when `randomCount > 0`):

| Field | Description |
|-------|-------------|
| Label | "Random Pets" |
| `randomPostCount` | Total posts linked to random pets in this family |

- Tap → navigates to **Random Pet Posts screen** for this family (all posts tagged to random pets)
- No avatar, no breed, no Story button

---

### 3. Parents Row & Bottom Sheet

```
Parents    [2]  >
```

- Shows label "Parents" + `parentCount` badge
- Tap → opens **Parents bottom sheet** (slides up from bottom, overlays the screen)
- No login required to view

**Parents bottom sheet:**
```
[drag handle]
Parents                          [× close]
─────────────────────────────────────────
[avatar]  Minh Dang              [✉]
          @minhdang
─────────────────────────────────────────
[avatar]  Cecilia Tran           [✉]
          @ceciliatran
```

- Each parent row shows: avatar + display name + tag (`@nickname`) + **Message icon `[✉]`** on the right
- Tap a parent row (anywhere except the Message icon) → close bottom sheet → navigate to **User Posts screen** for that user
- Tap **Message icon `[✉]`** → start a **DM** with that user: `StartThread (BI)` `{ senderType: USER (self), receiverType: USER }` → open the thread in the Messages screen's Thread View (screen_10). Reuses existing DM thread if one exists. Requires login → redirect to Login if not authenticated.
  - Message icon is **hidden on your own row** (can't DM yourself)
- Viewing the parent list: no login required

---

### 4. About Section

- Label: "ABOUT"
- Body: `family.bio` — free-text, multi-line
- Hidden entirely if `bio` is null or empty

---

### 5. Posts Section

**View toggle (top-right of Posts section header):**
- **List view** (default): post cards identical to Explore feed
- **Grid view**: 3-column thumbnail grid

The selected view persists for the session (client-side state).

---

#### 5a. List View

- Post cards follow the canonical definition in `screen_1_home_explore.md` → **Section: Post Card** — same fields, layout, interactions, and `...` menu logic
- Sorted by `createdAt` desc (newest first)
- Paginated: 10 posts per page, infinite scroll loads more

---

#### 5b. Grid View

- 3-column equal-width grid, no gaps (or minimal gap)
- Each cell shows the **thumbnail of the first media item** of the post:
  - `uploaded` image → `media[0].url`
  - `uploaded` video → `media[0].thumbnailUrl`
  - `embedded` → `media[0].thumbnailUrl`
- If a post has multiple media items, show a small multi-media indicator icon on the cell (top-right corner)
- Tap on any cell → navigates to **Post Detail screen** for that post
- Same pagination as list view (load more on scroll)

---

## API Endpoints Required

> FollowFamily and UnfollowFamily mutations are shared from Screen 1.

---

### Q. Query: `Family`

Fetch family profile data for the info card, pets list, and about section.

**Auth:** Optional — bearer token populates `social.isFollowedByMe`.

**Operation:**
```graphql
query Family($id: ID!) {
  family(id: $id) {
    id
    name
    tag
    location {
      city
      cityCode
      country
      countryCode
    }
    avatarUrl
    petAvatars
    petCount
    randomCount
    randomPostCount
    social {
      followersCount
      isFollowedByMe
    }
    familyType
    shortDescription
    donorCount
    bio
    parentCount
    pets {
      id
      name
      avatarUrl
      breed {
        nameVi
        nameEn
      }
      sex
      ageMonths
      postCount
    }
  }
}
```

**Variables:**
```json
{ "id": "fam_xyz" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "family": {
      "id": "fam_xyz",
      "name": "Minh's Family",
      "tag": "minhfamily",
      "location": {
        "city": "HCMC",
        "cityCode": "SGN",
        "country": "Vietnam",
        "countryCode": "VN"
      },
      "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
      "petAvatars": [
        "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
        "https://cdn.petapp.com/pets/pet_222/avatar.jpg"
      ],
      "petCount": 2,
      "randomCount": 10,
      "randomPostCount": 23,
      "social": {
        "followersCount": 287,
        "isFollowedByMe": false
      },
      "familyType": "CHARITY",
      "shortDescription": "Home-based cat rescue",
      "donorCount": 184,
      "bio": "Hi, I'm My 🐱 I take in stray and abandoned cats from the streets of HCMC.",
      "parentCount": 1,
      "pets": [
        {
          "id": "pet_111",
          "name": "Bụi",
          "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
          "breed": { "nameVi": "Mèo vằn cam", "nameEn": "Orange Tabby Cat" },
          "sex": "MALE",
          "ageMonths": 36,
          "postCount": 47
        },
        {
          "id": "pet_222",
          "name": "Chao",
          "avatarUrl": "https://cdn.petapp.com/pets/pet_222/avatar.jpg",
          "breed": { "nameVi": "Mèo nội địa Việt Nam", "nameEn": "Vietnamese Native" },
          "sex": "FEMALE",
          "ageMonths": 24,
          "postCount": 38
        }
      ]
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `FAMILY_NOT_FOUND` | Family does not exist (returned in `errors[]`) |

---

### R. Query: `FamilyFeed`

Fetch paginated posts for the family feed (used by both list view and grid view).

**Auth:** Optional — bearer token populates `isLoved` on each post.

**Operation:**
```graphql
query FamilyFeed($familyId: ID!, $first: Int = 20, $after: String) {
  familyFeed(familyId: $familyId, first: $first, after: $after) {
    edges {
      node {
        id
        family { id name avatarUrl familyType }
        author { id displayName avatarUrl }
        pets { id name avatarUrl }
        body
        location { city cityCode country countryCode }
        media {
          id type url thumbnailUrl mimeType
          width height durationSeconds provider
          mediaTag { type id species breed }
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
{ "familyId": "fam_xyz", "first": 10, "after": null }
```

**Response `200 OK`:**
```json
{
  "data": {
    "familyFeed": {
      "edges": [
        {
          "node": {
            "id": "post_abc",
            "family": { "id": "fam_xyz", "name": "Minh's Family", "avatarUrl": "...", "familyType": "STANDARD" },
            "author": { "id": "user_001", "displayName": "Minh Tuan", "avatarUrl": "..." },
            "pets": [{ "id": "pet_111", "name": "Bụi", "avatarUrl": "..." }],
            "body": "Bụi nằm chờ mama 🌕",
            "location": { "city": "Hồ Chí Minh", "cityCode": "HCM", "country": "Việt Nam", "countryCode": "VN" },
            "media": [
              {
                "id": "media_001", "type": "UPLOADED",
                "url": "https://cdn.petapp.com/media/001.jpg",
                "thumbnailUrl": null, "mimeType": "image/jpeg",
                "width": 1080, "height": 1080, "durationSeconds": null, "provider": null,
                "mediaTag": { "type": "PET", "id": "pet_111", "species": "Cat", "breed": "Orange Tabby Cat" }
              }
            ],
            "loveCount": 42, "commentsCount": 5,
            "isLoved": false, "visibility": "PUBLIC",
            "createdAt": "2026-06-06T08:00:00Z"
          }
        }
      ],
      "pageInfo": { "endCursor": "eyJ...", "hasNextPage": true }
    }
  }
}
```

> `edges[].node` follows the canonical Post shape from `screen_1_home_explore.md` → Query A (ExploreFeed).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `FAMILY_NOT_FOUND` | Family does not exist (returned in `errors[]`) |

> Same `Post` shape as Screen 1. Returns only posts the caller is allowed to see (privacy rules from Screen 1 apply). Sorted by `createdAt` desc.

---

### S. Query: `FamilyParents`

Fetch the list of users linked to this family (loaded when user taps the Parents row to open the bottom sheet).

**Auth:** Not required.

**Operation:**
```graphql
query FamilyParents($familyId: ID!) {
  familyParents(familyId: $familyId) {
    parents {
      id
      displayName
      username
      avatarUrl
    }
    totalCount
  }
}
```

**Variables:**
```json
{ "familyId": "fam_xyz" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "familyParents": {
      "parents": [
        {
          "id": "user_001",
          "displayName": "Minh Dang",
          "username": "minhdang",
          "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
        },
        {
          "id": "user_002",
          "displayName": "Cecilia Tran",
          "username": "ceciliatran",
          "avatarUrl": "https://cdn.petapp.com/users/user_002/avatar.jpg"
        }
      ],
      "totalCount": 2
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `FAMILY_NOT_FOUND` | Family does not exist (returned in `errors[]`) |

---

### T. Query: `PetPosts`

> ⚠️ Backend has no petFeed yet (GAP petapp-be#884) — kept as-is, pending backend.

Fetch posts for a specific named pet.  
**Auth:** Optional — bearer token enables `followers` + `private` posts for eligible viewers.

**Operation:**
```graphql
query PetPosts($petId: ID!, $cursor: String, $limit: Int) {
  petPosts(petId: $petId, cursor: $cursor, limit: $limit) {
    pet {
      id
      name
      breed {
        nameVi
        nameEn
      }
      species {
        name
        iconEmoji
      }
      avatarUrl
    }
    posts {
      ...PostCard
    }
    nextCursor
    hasMore
  }
}
```

**Variables:** `{ "petId": "pet_111", "limit": 10 }`

**Response `200 OK`:**
```json
{
  "data": {
    "petPosts": {
      "pet": { "id": "pet_111", "name": "Bụi", "species": { "name": "Cat", "iconEmoji": "🐱" }, "breed": { "nameVi": "Mèo vằn cam", "nameEn": "Orange Tabby Cat" }, "avatarUrl": "..." },
      "posts": [ { "id": "post_abc", "body": "Bụi nằm chờ mama 🌕", "media": [{ "mediaTag": { "type": "PET", "id": "pet_111", "species": "Cat", "breed": "Orange Tabby Cat" } }], "loveCount": 42, "commentsCount": 5, "visibility": "PUBLIC", "createdAt": "2026-06-06T08:00:00Z" } ],
      "nextCursor": "cursor_xyz", "hasMore": true
    }
  }
}
```

**Notes:**
- `posts[]` follows the canonical Post shape from screen_1 Query A (ExploreFeed).
- Server enforces privacy: unauthenticated → `public` only; authenticated → `public` + `followers` (if following family) + `private` (if family member).

**Errors:**

| Code | Scenario |
|------|----------|
| `PET_NOT_FOUND` | Pet does not exist or has been deleted |

---

### U. Query: `RandomPetPosts`

> ⚠️ **GAP petapp-be#886:** `randomPetPosts` is a feed scoped by `familyId` (only random-pet posts within one family). Backend `exploreFeed` is global cross-family and takes no `familyId`, so mapping to it would lose the family scope entirely — kept as-is, pending backend `randomFeed(familyId)`.

Fetch posts with random (unmatched) animal detections for a specific family.  
**Auth:** Optional — same privacy enforcement as above.

**Operation:**
```graphql
query RandomPetPosts($familyId: ID!, $cursor: String, $limit: Int) {
  randomPetPosts(familyId: $familyId, cursor: $cursor, limit: $limit) {
    family {
      id
      name
      avatarUrl
      randomCount
    }
    posts {
      ...PostCard
    }
    nextCursor
    hasMore
  }
}
```

**Variables:** `{ "familyId": "fam_xyz", "limit": 10 }`

**Response `200 OK`:**
```json
{
  "data": {
    "randomPetPosts": {
      "family": { "id": "fam_xyz", "name": "Minh's Family", "avatarUrl": "...", "randomCount": 10 },
      "posts": [ { "id": "post_def", "body": "Mèo lạ ghé thăm nhà 🐱", "media": [{ "mediaTag": { "type": "RANDOM", "id": null, "species": "Cat", "breed": "British Shorthair" } }], "loveCount": 18, "commentsCount": 2, "visibility": "PUBLIC", "createdAt": "2026-06-05T10:00:00Z" } ],
      "nextCursor": "cursor_abc", "hasMore": true
    }
  }
}
```

**Notes:**
- `posts[]` follows the canonical Post shape from screen_1 Query A (ExploreFeed).
- Server filters: only posts where at least one media has `mediaTag.type = "random" AND (breed IS NOT NULL OR species IS NOT NULL)`.
- `mediaTag.type = "random"` → no pet badge shown on any media frame (client applies existing rules).

**Errors:**

| Code | Scenario |
|------|----------|
| `FAMILY_NOT_FOUND` | Family does not exist |

---

### V. Query: `AuthorFeed`

Fetch posts authored by a specific user.  
**Auth:** Optional — same privacy enforcement as above.

**Operation:**
```graphql
query AuthorFeed($authorId: ID!, $first: Int = 20, $after: String) {
  authorFeed(authorId: $authorId, first: $first, after: $after) {
    edges {
      node {
        id
        family { id name avatarUrl familyType }
        author { id displayName avatarUrl }
        pets { id name avatarUrl }
        body
        location { city cityCode country countryCode }
        media {
          id type url thumbnailUrl mimeType
          width height durationSeconds provider
          mediaTag { type id species breed }
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

**Variables:** `{ "authorId": "user_001", "first": 10 }`

**Response `200 OK`:**
```json
{
  "data": {
    "authorFeed": {
      "edges": [
        {
          "node": {
            "id": "post_ghi",
            "body": "Bụi dậy sớm cùng tui 🌅",
            "media": [{ "mediaTag": { "type": "PET", "id": "pet_111", "species": "Cat", "breed": "Orange Tabby Cat" } }],
            "loveCount": 34, "commentsCount": 6,
            "visibility": "PUBLIC", "createdAt": "2026-06-06T07:00:00Z"
          }
        }
      ],
      "pageInfo": { "endCursor": null, "hasNextPage": false }
    }
  }
}
```

**Notes:**
- `edges[].node` follows the canonical Post shape from screen_1 Query A (ExploreFeed).

**Errors:**

| Code | Scenario |
|------|----------|
| `USER_NOT_FOUND` | User does not exist |

---

## User Flow Diagrams

### Open Family Posts screen

```
User taps family name on a post
  └─> Family query (Q) { familyId }
        ├─ FAMILY_NOT_FOUND → show "Family not found" error state
        └─ 200 → render info card + pets list + about
              └─> FamilyFeed query (R) { familyId, first: 10 }
                    └─> Render posts in default list view
```

### Switch to Grid View

```
User taps grid view icon
  └─> Re-render existing loaded posts as 3-column grid (no new API call)
        └─> Scroll to load more → FamilyFeed query (R) { familyId, after: <pageInfo.endCursor> }
```

### Tap Pet row

```
User taps a pet row
  └─> Navigate to Pet Posts screen for that pet (petId from row data)
```

### Tap Random Pets row

```
User taps Random Pets row  (only visible when randomCount > 0)
  └─> Navigate to Random Pet Posts screen for this family (familyId)
```

### Tap Parents row → bottom sheet

```
User taps Parents row
  └─> Family query (Q) (parents already loaded)
        └─> Open Parents bottom sheet with list of parents
              └─> User taps a parent row
                    └─> Close bottom sheet → Navigate to User Posts screen (userId)
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| `petCount = 0` | Show default family avatar; no pets list section |
| `petCount = 1` | Show single pet avatar (no stacking animation) |
| `petCount ≥ 2` | Stack up to 5 pet avatars; rotate through them on timer (client-side only) |
| `familyType = CHARITY` | Show "CHARITY" ribbon on avatar; show Charity Section (shortDescription, donorCount, Donate button) |
| `familyType = STANDARD` | No ribbon; no Charity Section |
| `petCount ≤ 5` | Pets list: single vertical column |
| `petCount > 5` | Pets list: horizontal scroll, 5 rows per column, next column peeks on right edge |
| `bio` is null or empty | Hide About section entirely |
| `randomCount = 0` | Hide "randoms" from stats line; hide Random Pets row from pets list |
| `randomCount > 0` | Show "N randoms" in stats line; show Random Pets row at bottom of pets list |
| Story button | Hidden — Story (`screen_29`) is member-only; this screen's viewer is always a non-member |
| Follow button — not logged in | Tap → redirect to Login |
| Message button — not logged in | Tap → redirect to Login |
| No posts | Show empty state: "No posts yet" |
| Grid view — post has multiple media | Show thumbnail of `media[0]`; overlay small multi-media icon top-right of cell |
| Grid view — tap cell | Navigate to Post Detail screen |
| Privacy rules | Same as Explore — unauthenticated sees `public` posts only |

---

## Linked Screens

All three linked screens follow the same pattern: **profile header + posts list**.  
Posts list is identical to the Family Posts list view — same card format as Explore, 10 per page, infinite scroll, list/grid toggle.  
All canonical post card tap interactions apply (see `screen_1_home_explore.md` → Post Card):
- Tap **family name** → Family Posts screen
- Tap **author name** → User Posts screen
- Tap **pet badge** on media → Pet Posts screen

---

### Pet Posts Screen

**Triggered from:**
- Tap **pet badge** on any post card media (Explore, Post Detail, Family Posts, User Posts, Random Pet Posts, Loved Posts) — any viewer
- Tap **pet row** in Family Posts — viewer is always unauth/non-member

**Header:**

| Field | Display |
|-------|---------|
| `avatarUrl` | Pet avatar |
| `name` | Pet name |
| `breed` | Breed name |

**Posts:**
- No filter tabs; no Suggested Families widget
- Sorted `createdAt` desc (newest first)
- 10 posts per page, infinite scroll
- Privacy rules: same server-side enforcement as Explore — unauthenticated sees `public` posts only; authenticated sees `public` + `followers` (if following) + `private` (if family member)
- Empty state: possible if pet has no posts visible to this viewer (e.g. all posts are private) — show message "No posts yet"
- All canonical post card tap interactions apply (see `screen_1_home_explore.md` → Post Card)
- API: Query `PetPosts` (endpoint T above)

---

### Random Pet Posts Screen

**Triggered from:**
- Tap **Random Pets row** in Family Posts (only when `randomCount > 0`)
- Tap **"View All →"** in Random Pets section in My Pets (screen_8; only shown when random posts exist)

**Header:**

| Field | Display |
|-------|---------|
| Family avatar | Family avatar |
| Label | `"Random Pets"` |
| `randomCount` | e.g. `"10 randoms"` |

**Posts:**
- Scope: per-family only — all posts from this family where at least one media has `mediaTag.type = "random" AND (breed IS NOT NULL OR species IS NOT NULL)`
- No filter tabs; no Suggested Families widget
- Sorted `createdAt` desc (newest first)
- 10 posts per page, infinite scroll
- Privacy rules: same server-side enforcement as Explore
- Empty state: not applicable — both entry points are guarded (`randomCount > 0` / section hidden when 0 posts)
- Post card follows canonical format (see `screen_1_home_explore.md` → Post Card); `mediaTag.type = "random"` → no pet badge shown on any media frame → no link to Pet Posts screen from within this screen
- API: Query `RandomPetPosts` (endpoint U above)

---

### User Posts Screen

**Triggered from:**
- Tap **author name** on any post card (Explore, Post Detail, Family Posts, Pet Posts, Random Pet Posts, Loved Posts)
- Tap **parent name** in the Parents bottom sheet (Family Posts, My Pets)

**Header:**

| Field | Display |
|-------|---------|
| `avatarUrl` | User avatar |
| `displayName` | User's display name |
| `username` | `@username` |

**Posts:**
- No filter tabs; no Suggested Families widget
- Sorted `createdAt` desc (newest first)
- 10 posts per page, infinite scroll
- Privacy rules: same server-side enforcement as Explore — unauthenticated sees `public` posts only; authenticated sees `public` + `followers` (if following the post's family) + `private` (if family member). Viewer always sees at least the post they tapped from.
- Empty state: not applicable — screen is only reachable by tapping a post card, so at least 1 post is always visible
- All canonical post card tap interactions apply (see `screen_1_home_explore.md` → Post Card)
- API: Query `AuthorFeed` (endpoint V above)

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Post count shown in header | Not shown |
| 2 | Pet list pagination | None — all pets shown at once |
| 3 | Default post view | List view |
| 4 | Story button | Not shown here — pet Story (`screen_29`) is member-only; Family Posts viewer is always a non-member |
| 5 | `randomCount = 0` display | Hide from stats line and hide Random Pets row |
| 6 | Parents navigation | Bottom sheet popup (not a separate screen) |
| 7 | Random pets navigation | Separate Random Pet Posts screen |
| 8 | Donate button (interim) | No wallet yet — opens a **chat** with the charity, pre-filled with a Vietnamese support message (same mechanic as "I saw" / "Inquire to Adopt"); `donorCount` not yet driven by it |
| 9 | Pets list layout threshold | ≤ 5 pets: vertical list; > 5 pets: horizontal scroll, 5 per column |

---

## Open Items (next steps)

- **Donate — wallet / real donations**: the interim Donate opens a chat only. A later phase adds an actual donation flow (in-app wallet / transfer), at which point `donorCount` is driven by completed donations and a **donate count** (how many people tapped Donate / donated, akin to rescue `inquiriesCount`) can be tracked. Not built this phase.
