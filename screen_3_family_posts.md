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
  ├── Charity Section  ← only shown when type = charity
  │     ├── [♡ icon]  short_description
  │     ├── [♡ icon]  "N people have helped"
  │     └── [Donate button — full width]
  │
  ├── Pets List  ← horizontal scroll when pet_count > 5
  │     Column 1 (pets 1–5)    Column 2 (pets 6–10)    ...
  │     ├── Pet row 1           ├── Pet row 6
  │     ├── Pet row 2           ├── Pet row 7
  │     ├── Pet row 3           ├── Pet row 8
  │     ├── Pet row 4           ├── Pet row 9
  │     └── Pet row 5           └── Pet row 10
  │     [Random Pets row — only if random_count > 0, always in last column]
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
| `avatar_url` | Used as fallback / single avatar |
| `pet_avatars` | Ordered list of pet avatar URLs for the stacked display (up to 5) |
| `pet_count` | Number of actual pets in the family |
| `random_count` | Number of media items where `media_tag.type = "random" AND breed IS NOT NULL` — AI detected a breed but could not match to a named pet. Does NOT count media with no breed detected. Hidden from stats line when `0`. |
| `random_post_count` | Total posts linked to random pets in this family. Used in the Random Pets row. |
| `follower_count` | Total followers |
| `type` | `standard` \| `charity` |
| `is_following` | Boolean (false when unauthenticated) |
| `about` | Free-text description |
| `short_description` | Short tagline for charity families (e.g. `"Home-based cat rescue"`). `null` for standard families. |
| `donor_count` | Number of people who have donated. Only meaningful when `type = charity`; `null` for standard families. |

**Avatar display rules:**
- `pet_count = 0` → show single default family avatar
- `pet_count = 1` → show that pet's avatar
- `pet_count ≥ 2` → show stacked overlapping pet avatars (up to 5); auto-rotates through them on a timer (client-side animation only, no API call)
- `type = charity` → show a **"CHARITY" ribbon badge** overlaid on the avatar (top-left corner)

**Follow button:**
- Not following → button label "Follow"; tap → `POST /families/{id}/follow` (requires login → redirect to Login)
- Following → button label "Following"; tap → `DELETE /families/{id}/follow`
- Reuse endpoints D and E from Screen 1

**Message button:**
- Tap → opens Messages screen for this family (requires login → redirect to Login if not authenticated)

---

### 2. Charity Section *(only rendered when `type = charity`)*

Displayed between the Family Info Card and the Pets List.

| Element | Description |
|---------|-------------|
| `short_description` | Displayed with a ♡ icon prefix, e.g. `"♡ Home-based cat rescue"` |
| `donor_count` | Displayed as `"♡ 184 people have helped"` |
| Donate button | Full-width button; tap → in-app Donate screen (currently "Coming Soon") |

- The entire section is hidden for `standard` families.
- `donor_count` label updates after a successful donation (handled by Donate screen, not this screen).

---

### 3. Pets List

**Layout rules:**
- `pet_count ≤ 5` → single vertical column (current behaviour)
- `pet_count > 5` → **horizontal scroll**, 5 rows per column; swipe left to reveal more columns
  - Column 1: pets 1–5, Column 2: pets 6–10, etc.
  - Next column's avatars peek slightly on the right edge to indicate scrollability
  - Random Pets row (if applicable) always appears as the last item in the final column

Each pet row:

| Field | Description |
|-------|-------------|
| `id` | Pet ID |
| `avatar_url` | Pet avatar |
| `name` | Pet name |
| `breed` | Breed name (may be truncated with `...` if long) |
| `gender` | `male` \| `female` \| `unknown` |
| `age_display` | Human-readable age string, e.g. `"3 years"`, `"5 months"` |
| `post_count` | Total posts linked to this pet |

**Interactions:**
- Tap anywhere on the row (except Story button) → navigates to **Pet Posts screen** for that pet (viewer is always unauth/non-member on this screen)
- **Story button** → disabled for now (render as disabled state, no action)

**Random Pets row** (shown only when `random_count > 0`):

| Field | Description |
|-------|-------------|
| Label | "Random Pets" |
| `random_post_count` | Total posts linked to random pets in this family |

- Tap → navigates to **Random Pet Posts screen** for this family (all posts tagged to random pets)
- No avatar, no breed, no Story button

---

### 3. Parents Row & Bottom Sheet

```
Parents    [2]  >
```

- Shows label "Parents" + `parent_count` badge
- Tap → opens **Parents bottom sheet** (slides up from bottom, overlays the screen)
- No login required to view

**Parents bottom sheet:**
```
[drag handle]
Parents                          [× close]
─────────────────────────────────────────
[avatar]  Minh Dang
          @minhdang
─────────────────────────────────────────
[avatar]  Cecilia Tran
          @ceciliatran
```

- Each parent row shows: avatar + display name + tag (`@nickname`)
- Tap a parent row → close bottom sheet → navigate to **User Posts screen** for that user
- No login required to view

---

### 4. About Section

- Label: "ABOUT"
- Body: `family.about` — free-text, multi-line
- Hidden entirely if `about` is null or empty

---

### 5. Posts Section

**View toggle (top-right of Posts section header):**
- **List view** (default): post cards identical to Explore feed
- **Grid view**: 3-column thumbnail grid

The selected view persists for the session (client-side state).

---

#### 5a. List View

- Post cards follow the canonical definition in `screen_1_home_explore.md` → **Section: Post Card** — same fields, layout, interactions, and `...` menu logic
- Sorted by `created_at` desc (newest first)
- Paginated: 10 posts per page, infinite scroll loads more

---

#### 5b. Grid View

- 3-column equal-width grid, no gaps (or minimal gap)
- Each cell shows the **thumbnail of the first media item** of the post:
  - `uploaded` image → `media[0].url`
  - `uploaded` video → `media[0].thumbnail_url`
  - `embedded` → `media[0].thumbnail_url`
- If a post has multiple media items, show a small multi-media indicator icon on the cell (top-right corner)
- Tap on any cell → navigates to **Post Detail screen** for that post
- Same pagination as list view (load more on scroll)

---

## API Endpoints Required

> FollowFamily and UnfollowFamily mutations are shared from Screen 1.

---

### Q. Query: `Family`

Fetch family profile data for the info card, pets list, and about section.

**Auth:** Optional — bearer token populates `isFollowing`.

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
    followerCount
    type
    isFollowing
    shortDescription
    donorCount
    about
    parentCount
    pets {
      id
      name
      avatarUrl
      breed
      gender
      ageDisplay
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
      "followerCount": 287,
      "type": "CHARITY",
      "isFollowing": false,
      "shortDescription": "Home-based cat rescue",
      "donorCount": 184,
      "about": "Hi, I'm My 🐱 I take in stray and abandoned cats from the streets of HCMC.",
      "parentCount": 1,
      "pets": [
        {
          "id": "pet_111",
          "name": "Bụi",
          "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
          "breed": "Orange Tabby Cat",
          "gender": "MALE",
          "ageDisplay": "3 years",
          "postCount": 47
        },
        {
          "id": "pet_222",
          "name": "Chao",
          "avatarUrl": "https://cdn.petapp.com/pets/pet_222/avatar.jpg",
          "breed": "Vietnamese Native",
          "gender": "FEMALE",
          "ageDisplay": "2 years",
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

### R. Query: `FamilyPosts`

Fetch paginated posts for the family feed (used by both list view and grid view).

**Auth:** Optional — bearer token populates `isLoved` on each post.

**Operation:**
```graphql
query FamilyPosts($familyId: ID!, $cursor: String, $limit: Int) {
  familyPosts(familyId: $familyId, cursor: $cursor, limit: $limit) {
    posts {
      ...Post
    }
    nextCursor
    hasMore
  }
}
```

**Variables:**
```json
{ "familyId": "fam_xyz", "cursor": null, "limit": 10 }
```

**Response `200 OK`:**
```json
{
  "data": {
    "familyPosts": {
      "posts": [],
      "nextCursor": "cursor_abc123",
      "hasMore": true
    }
  }
}
```

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
      tag
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
          "tag": "minhdang",
          "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
        },
        {
          "id": "user_002",
          "displayName": "Cecilia Tran",
          "tag": "ceciliatran",
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

## User Flow Diagrams

### Open Family Posts screen

```
User taps family name on a post
  └─> GET /families/{family_id}
        ├─ 404 → show "Family not found" error state
        └─ 200 → render info card + pets list + about
              └─> GET /families/{family_id}/posts?limit=10
                    └─> Render posts in default list view
```

### Switch to Grid View

```
User taps grid view icon
  └─> Re-render existing loaded posts as 3-column grid (no new API call)
        └─> Scroll to load more → GET /families/{family_id}/posts?cursor=<next_cursor>
```

### Tap Pet row

```
User taps a pet row
  └─> Navigate to Pet Posts screen for that pet (pet_id from row data)
```

### Tap Random Pets row

```
User taps Random Pets row  (only visible when random_count > 0)
  └─> Navigate to Random Pet Posts screen for this family (family_id)
```

### Tap Parents row → bottom sheet

```
User taps Parents row
  └─> GET /families/{family_id}/parents
        └─> Open Parents bottom sheet with list of parents
              └─> User taps a parent row
                    └─> Close bottom sheet → Navigate to User Posts screen (user_id)
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| `pet_count = 0` | Show default family avatar; no pets list section |
| `pet_count = 1` | Show single pet avatar (no stacking animation) |
| `pet_count ≥ 2` | Stack up to 5 pet avatars; rotate through them on timer (client-side only) |
| `type = charity` | Show "CHARITY" ribbon on avatar; show Charity Section (short_description, donor_count, Donate button) |
| `type = standard` | No ribbon; no Charity Section |
| `pet_count ≤ 5` | Pets list: single vertical column |
| `pet_count > 5` | Pets list: horizontal scroll, 5 rows per column, next column peeks on right edge |
| `about` is null or empty | Hide About section entirely |
| `random_count = 0` | Hide "randoms" from stats line; hide Random Pets row from pets list |
| `random_count > 0` | Show "N randoms" in stats line; show Random Pets row at bottom of pets list |
| Story button | Render as disabled (greyed out); no tap action |
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
| `avatar_url` | Pet avatar |
| `name` | Pet name |
| `breed` | Breed name |

**Posts:**
- No filter tabs; no Suggested Families widget
- Sorted `created_at` desc (newest first)
- 10 posts per page, infinite scroll
- Privacy rules: same server-side enforcement as Explore — unauthenticated sees `public` posts only; authenticated sees `public` + `followers` (if following) + `private` (if family member)
- Empty state: possible if pet has no posts visible to this viewer (e.g. all posts are private) — show message "No posts yet"
- All canonical post card tap interactions apply (see `screen_1_home_explore.md` → Post Card)
- API: `GET /pets/{pet_id}/posts?cursor=&limit=10`

---

### Random Pet Posts Screen

**Triggered from:**
- Tap **Random Pets row** in Family Posts (only when `random_count > 0`)
- Tap **"View All →"** in Random Pets section in My Pets (screen_8; only shown when random posts exist)

**Header:**

| Field | Display |
|-------|---------|
| Family avatar | Family avatar |
| Label | `"Random Pets"` |
| `random_count` | e.g. `"10 randoms"` |

**Posts:**
- Scope: per-family only — all posts from this family where at least one media has `media_tag.type = "random" AND breed IS NOT NULL`
- No filter tabs; no Suggested Families widget
- Sorted `created_at` desc (newest first)
- 10 posts per page, infinite scroll
- Privacy rules: same server-side enforcement as Explore
- Empty state: not applicable — both entry points are guarded (`random_count > 0` / section hidden when 0 posts)
- Post card follows canonical format (see `screen_1_home_explore.md` → Post Card); `media_tag.type = "random"` → no pet badge shown on any media frame → no link to Pet Posts screen from within this screen
- API: `GET /families/{family_id}/posts?type=random&cursor=&limit=10`

---

### User Posts Screen

**Triggered from:**
- Tap **author name** on any post card (Explore, Post Detail, Family Posts, Pet Posts, Random Pet Posts, Loved Posts)
- Tap **parent name** in the Parents bottom sheet (Family Posts, My Pets)

**Header:**

| Field | Display |
|-------|---------|
| `avatar_url` | User avatar |
| `display_name` | User's display name |
| `tag` | `@tag` |

**Posts:**
- No filter tabs; no Suggested Families widget
- Sorted `created_at` desc (newest first)
- 10 posts per page, infinite scroll
- Privacy rules: same server-side enforcement as Explore — unauthenticated sees `public` posts only; authenticated sees `public` + `followers` (if following the post's family) + `private` (if family member). Viewer always sees at least the post they tapped from.
- Empty state: not applicable — screen is only reachable by tapping a post card, so at least 1 post is always visible
- All canonical post card tap interactions apply (see `screen_1_home_explore.md` → Post Card)
- API: `GET /users/{user_id}/posts?cursor=&limit=10`

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Post count shown in header | Not shown |
| 2 | Pet list pagination | None — all pets shown at once |
| 3 | Default post view | List view |
| 4 | Story button | Disabled (coming later) |
| 5 | `random_count = 0` display | Hide from stats line and hide Random Pets row |
| 6 | Parents navigation | Bottom sheet popup (not a separate screen) |
| 7 | Random pets navigation | Separate Random Pet Posts screen |
| 8 | Donate button destination | In-app Donate screen (currently "Coming Soon") |
| 9 | Pets list layout threshold | ≤ 5 pets: vertical list; > 5 pets: horizontal scroll, 5 per column |
