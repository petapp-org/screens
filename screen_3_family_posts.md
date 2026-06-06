# Screen 3: Family Posts

## Overview

Profile and post feed for a single family.  
Navigated to from: tapping a family name on any post card (Explore feed, Post Detail, etc.).  
Accessible without login — unauthenticated users can view everything. Actions (Follow, Message) require login.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: Family name (static)

[Scrollable area]
  ├── Family Info Card
  │     ├── Avatar (stacked pet photos or default)
  │     ├── Family name
  │     ├── @handle  ·  City, Country
  │     ├── N pets  [· N randoms — only if > 0]  · N followers
  │     └── [Follow button]  [Message button]
  │
  ├── Pets List
  │     ├── Pet row 1: avatar · name · breed · gender · age · N posts  [Story — disabled]  [>]
  │     ├── Pet row 2: ...
  │     └── [Random Pets row — only if random_count > 0]
  │           "Random Pets"  ·  N posts  [>]
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
| `handle` | Unique identifier, prefixed with `@` (e.g. `@minhfamily`) |
| `city` | City name |
| `country` | Country code or name (e.g. `VN`) |
| `avatar_url` | Used as fallback / single avatar |
| `pet_avatars` | Ordered list of pet avatar URLs for the stacked display (up to 5) |
| `pet_count` | Number of actual pets in the family |
| `random_count` | Number of "randoms" — pets tagged to a breed but not a named pet of this family. Hidden from stats line when `0`. |
| `random_post_count` | Total posts linked to random pets in this family. Used in the Random Pets row. |
| `follower_count` | Total followers |
| `type` | `standard` \| `charity` |
| `is_following` | Boolean (false when unauthenticated) |
| `about` | Free-text description |

**Avatar display rules:**
- `pet_count = 0` → show single default family avatar
- `pet_count = 1` → show that pet's avatar
- `pet_count ≥ 2` → show stacked overlapping pet avatars (up to 5); auto-rotates through them on a timer (client-side animation only, no API call)

**Follow button:**
- Not following → button label "Follow"; tap → `POST /families/{id}/follow` (requires login → redirect to Login)
- Following → button label "Following"; tap → `DELETE /families/{id}/follow`
- Reuse endpoints D and E from Screen 1

**Message button:**
- Tap → opens Messages screen for this family (requires login → redirect to Login if not authenticated)

---

### 2. Pets List

All pets belonging to this family, shown in a vertical list (no pagination).

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
- Tap anywhere on the row (except Story button) → navigates to **Pet Posts screen** for that pet
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

- Each parent row shows: avatar + display name + handle (`@nickname`)
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

- Post cards identical to Screen 1 (Explore) — same fields, same interactions, same `...` menu logic
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

> Follow/unfollow endpoints (D, E) are shared from Screen 1.

---

### Q. `GET /families/{family_id}`

Fetch family profile data for the info card, pets list, and about section.

**Headers:**
- `Authorization: Bearer <token>` — optional. Populates `is_following`.

**Response `200 OK`:**

```json
{
  "id": "fam_xyz",
  "name": "Minh's Family",
  "handle": "minhfamily",
  "city": "HCMC",
  "country": "VN",
  "avatar_url": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
  "pet_avatars": [
    "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
    "https://cdn.petapp.com/pets/pet_222/avatar.jpg"
  ],
  "pet_count": 2,
  "random_count": 10,
  "random_post_count": 23,
  "follower_count": 287,
  "type": "standard",
  "is_following": false,
  "about": "A family of three — a cat, a dog, and a pony. Sharing the small everyday moments. 🐾",
  "parent_count": 2,
  "pets": [
    {
      "id": "pet_111",
      "name": "Bụi",
      "avatar_url": "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
      "breed": "Orange Tabby Cat",
      "gender": "male",
      "age_display": "3 years",
      "post_count": 47
    },
    {
      "id": "pet_222",
      "name": "Chao",
      "avatar_url": "https://cdn.petapp.com/pets/pet_222/avatar.jpg",
      "breed": "Vietnamese Native",
      "gender": "female",
      "age_display": "2 years",
      "post_count": 38
    }
  ]
}
```

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `FAMILY_NOT_FOUND` | Family does not exist |

---

### R. `GET /families/{family_id}/posts`

Fetch paginated posts for the family feed (used by both list view and grid view).

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `cursor` | string | No | — | Pagination cursor |
| `limit` | int | No | `10` | Posts per page |

**Headers:**
- `Authorization: Bearer <token>` — optional. Populates `is_loved` on each post.

**Response `200 OK`:** Same shape as `GET /feed/explore` — `posts` array + `next_cursor` + `has_more`.

**Notes:**
- Returns only posts the caller is allowed to see (privacy rules from Screen 1 apply)
- Sorted by `created_at` desc

---

### S. `GET /families/{family_id}/parents`

Fetch the list of users linked to this family (loaded when user taps the Parents row to open the bottom sheet).

**Response `200 OK`:**
```json
{
  "parents": [
    {
      "id": "user_001",
      "display_name": "Minh Dang",
      "handle": "minhdang",
      "avatar_url": "https://cdn.petapp.com/users/user_001/avatar.jpg"
    },
    {
      "id": "user_002",
      "display_name": "Cecilia Tran",
      "handle": "ceciliatran",
      "avatar_url": "https://cdn.petapp.com/users/user_002/avatar.jpg"
    }
  ],
  "total_count": 2
}
```

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

## Linked Screens (to be specced separately)

| Screen | Triggered from |
|--------|---------------|
| Pet Posts screen | Tap a named pet row |
| Random Pet Posts screen | Tap the Random Pets row |
| User Posts screen | Tap a parent in the bottom sheet |

**User Posts screen — preview:**  
Shows a user's profile header (display name, `@handle`, which family/families they belong to) followed by all posts that user has created, same card format as Explore, 10 per page.

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
