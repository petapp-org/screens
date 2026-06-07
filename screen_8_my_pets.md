# Screen 8: My Pets Tab

## Overview

The "My Pets" tab — first tab in the bottom navigation. Shows the user's **active family** with its pets, management tools, and random pet media.  
Requires login. If not logged in → redirect to Login.  
If logged in but no active family → show empty state with prompt to create or activate a family.

---

## UI Layout

```
[Header]
  Left: "My Pets" (title)
  Right: Search icon | Messages icon (red dot if unread) | Profile avatar

[Scrollable area]
  ├── Family Info Card (active family)
  │     ├── Avatar (stacked pet photos, rotates)
  │     ├── Family name
  │     ├── @tag  ·  📍 city_code, country_code
  │     ├── N pets · N randoms · N followers
  │     └── [Post button]  [Edit button]
  │
  ├── Pet Rows (one per named pet, no pagination)
  │     ├── [avatar]  Name · Breed
  │     │             gender · age · N posts
  │     │             [health status badge]
  │     │             [Story — disabled]  [>]
  │     └── (repeat for each pet)
  │
  ├── Manage Parents row  [>]
  │     └── → opens Parents bottom sheet
  │
  ├── RANDOM PETS section
  │     ├── "RANDOM PETS"  [+]
  │     ├── 2×2 grid preview (4 cells max)
  │     │     Each cell: thumbnail + breed name + 📍 location · time
  │     └── [View All →]  (shown if more than 4 items exist)
  │
  ├── MY PET POSTS section
  │     ├── "MY PET POSTS"  [Grid icon]  [List icon]
  │     └── Post list — 10 posts, infinite scroll (same as Explore)
  │
  └── (bottom padding)
```

---

## Components

### 1. Family Info Card

Displays the user's **active family**. Same data as Screen 3 (Family Posts) info card, but with owner-specific actions.

| Field | Display |
|-------|---------|
| `avatar_url` / `pet_avatars` | Stacked pet avatars (same rules as Screen 3: 0 pet = default, 1 = single, ≥2 = stacked rotating) |
| `name` | Family display name |
| `tag` | `@tag` |
| `city_code`, `country_code` | Displayed as `📍 HCM, VN` |
| `pet_count` | `N pets` |
| `random_count` | `N randoms` — hidden if 0 |
| `follower_count` | `N followers` |

**Post button:**
- Tap → navigate to **Create Post screen** (Screen 7) with this family pre-selected
- Available to all family members (owner + parents)

**Edit button:**
- Tap → navigate to **Update Family screen** (Screen 6) for the active family
- Available to **owner only** — hidden or disabled for parents

---

### 2. Pet Rows

All named pets of the active family — **both public and private** (viewer is always a family member here). No pagination — all shown at once.

Each row:

| Field | Display |
|-------|---------|
| `avatar_url` | Pet avatar |
| `is_public` | 🔒 lock icon shown on row if `is_public = false` (private pet) |
| `name` | Pet name |
| `breed` | Breed name (truncated with `...` if long) |
| `gender` | `Male` / `Female` / `Unknown` |
| `age_display` | e.g. `"3 years"`, `"5 months"` |
| `post_count` | `N posts` |
| `health_status` | Badge — see Health Status below |

**Health Status badge:**

| Status | Badge | Color | Meaning |
|--------|-------|-------|---------|
| `normal` | `NORMAL` | Green | No issues |
| `check` | `CHECK` | Amber/Orange | Needs attention |

> Full health status logic will be defined in a separate Pet Detail / Health spec.

**Story button:** disabled (greyed out, no action) — same as Screen 3.

**Tap row (anywhere except Story):** → navigate to **Pet Detail screen** for that pet.

---

### 3. Manage Parents Row

```
👤  Manage Parents                          [>]
```

- Tap → opens **Parents bottom sheet** (slides up from bottom)
- Available to owner only: owner can remove parents and cancel invites
- Parents who are not the owner see a read-only version (no Remove/Cancel actions)

**Parents bottom sheet:**

```
[drag handle]
Parents                                    [× close]
──────────────────────────────────────────────────
[avatar]  Minh Dang   [YOU]  [OWNER]      (no action)
──────────────────────────────────────────────────
[avatar]  Cecilia Tran  [PARENT]           [Remove]
──────────────────────────────────────────────────
[avatar]  Thao Nguyen  [INVITED]           [Cancel]
──────────────────────────────────────────────────
[+]  Invite Another Parent                 [>]
```

**Row types:**

| Row | Badges | Action | Who can see action |
|-----|--------|--------|--------------------|
| Owner (self) | `YOU` + `OWNER` | None | — |
| Accepted parent | `PARENT` | Remove | Owner only |
| Pending invite | `INVITED` | Cancel | Owner only |

**Remove action:**
- Confirmation dialog → `DELETE /families/{id}/parents/{user_id}` (endpoint AS from Screen 6)
- Row removed immediately on confirm

**Cancel invite action:**
- `DELETE /families/{id}/invites/{user_id}` (endpoint AR from Screen 6)
- Row removed immediately

**Invite Another Parent:**
- Opens invite search modal (same as Screen 6 → Section: Invite Search Modal)
- Uses endpoint AP (search users) + AQ (send invite) from Screen 6

> This bottom sheet shares the same data model and all API endpoints with Screen 6 (Parents section). No new endpoints needed.

---

### 4. Random Pets Section

Displays posts where at least one media has `media_tag.type = "random" AND (breed IS NOT NULL OR species IS NOT NULL)` — AI detected an animal but no named pet was matched.

**Preview grid (2×2, max 4 cells):**

Each cell shows:
- Media thumbnail (top)
- `breed` name if available, otherwise `species` name — in bold (below thumbnail)
- `📍 city_code, country_code · time` — time follows the same display rules as post cards (see `screen_1_home_explore.md` → Post Card → Time display rules). E.g. `"HCMC, VN · 3h"`, `"HCMC, VN · 2d"`, `"HCMC, VN · 28 May"`

- Tap any cell → **Post Detail screen** for that post
- If total breed-tagged posts > 4: show **"View All →"** link below grid
- **"View All →"** → **Random Pet Posts screen** (same pattern as Pet Posts screen in Screen 3, but filtered to posts with `media_tag.type = "random" AND breed IS NOT NULL`; header title: "Random Pets")
- If 0 breed-tagged posts → hide section entirely

**`[+]` button:**
- Tap → **Create Post screen** (Screen 7)
- In this context, AI scan **skips the family pet matching step** — result can only be `breed` or `random`, never `pet`. This means the scan API is called without `family_id` (or with a flag `skip_pet_match: true`)

---

### 5. My Pet Posts Section

All posts belonging to the active family, same as Family Posts screen (Screen 3).

- Section header: **"MY PET POSTS"** + list/grid view toggle
- Default: list view — canonical post card (Screen 1); all canonical tap interactions apply: tap **family name** → Family Posts, tap **author name** → User Posts, tap **pet badge** → Pet Posts
- Grid view: same 3-column thumbnail grid as Screen 3
- Sorted `created_at` desc (newest first)
- 10 posts per page, infinite scroll
- API: `GET /families/{family_id}/posts?cursor=&limit=10` (endpoint R from Screen 3)

---

## API Endpoints Required

> Most data for this screen reuses existing endpoints. New endpoints listed below. All calls go to `POST /graphql`.

### AY. Query: `ActiveFamily`

Fetch the active family with full detail for the My Pets screen.  
**Auth:** Required

**Operation:**
```graphql
query ActiveFamily {
  activeFamily {
    id
    name
    tag
    city
    cityCode
    country
    countryCode
    avatarUrl
    petAvatars
    petCount
    randomCount
    followerCount
    viewerRole
    pets {
      id
      name
      avatarUrl
      breed
      gender
      ageDisplay
      postCount
      healthStatus
    }
    parents {
      id
      name
      tag
      avatarUrl
      role
      status
    }
  }
}
```

**Variables:**
```json
{}
```

**Response `200 OK`:**
```json
{
  "data": {
    "activeFamily": {
      "id": "fam_xyz",
      "name": "Minh's Family",
      "tag": "minhfamily",
      "city": "Hồ Chí Minh",
      "cityCode": "HCM",
      "country": "Việt Nam",
      "countryCode": "VN",
      "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
      "petAvatars": ["..."],
      "petCount": 3,
      "randomCount": 10,
      "followerCount": 287,
      "viewerRole": "OWNER",
      "pets": [
        {
          "id": "pet_111",
          "name": "Bụi",
          "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
          "breed": "Orange Tabby Cat",
          "gender": "MALE",
          "ageDisplay": "3 years",
          "postCount": 47,
          "healthStatus": "NORMAL"
        },
        {
          "id": "pet_222",
          "name": "Măng",
          "avatarUrl": "https://cdn.petapp.com/pets/pet_222/avatar.jpg",
          "breed": "Buckskin Pony",
          "gender": "FEMALE",
          "ageDisplay": "5 years",
          "postCount": 24,
          "healthStatus": "CONCERN"
        }
      ],
      "parents": [
        { "id": "user_001", "name": "Minh Dang", "tag": "minhdang", "avatarUrl": "...", "role": "OWNER", "status": "JOINED" },
        { "id": "user_002", "name": "Cecilia Tran", "tag": "ceciliatran", "avatarUrl": "...", "role": "PARENT", "status": "JOINED" },
        { "id": "user_003", "name": "Thao Nguyen", "tag": "thaonguyen", "avatarUrl": "...", "role": "PARENT", "status": "INVITED" }
      ]
    }
  }
}
```

**Notes:**
- `viewerRole`: `OWNER` | `PARENT` — controls visibility of Edit button and management actions in Parents popup
- `parents` included here to avoid a separate API call when opening the bottom sheet

---

### AZ. Query: `FamilyRandomMedia`

Fetch breed-tagged media for the Random Pets section preview (2×2 grid + View All list).  
**Auth:** Required

**Operation:**
```graphql
query FamilyRandomMedia($familyId: ID!, $cursor: String, $limit: Int) {
  familyRandomMedia(familyId: $familyId, cursor: $cursor, limit: $limit) {
    media {
      id
      thumbnailUrl
      postId
      breed
      cityCode
      countryCode
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
{
  "familyId": "fam_xyz",
  "cursor": null,
  "limit": 20
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "familyRandomMedia": {
      "media": [
        {
          "id": "media_003",
          "thumbnailUrl": "https://cdn.petapp.com/media/003_thumb.jpg",
          "postId": "post_abc",
          "breed": "British Shorthair",
          "cityCode": "HCM",
          "countryCode": "VN",
          "createdAt": "2026-05-01T10:00:00Z"
        }
      ],
      "totalCount": 12,
      "nextCursor": "...",
      "hasMore": true
    }
  }
}
```

**Notes:**
- `thumbnailUrl` used directly in the 2×2 grid cells
- `breed`, `cityCode`, `countryCode`, `createdAt` used for cell subtitle display: `breed name · 📍 HCM, VN · 1 mo`
- Screen fetches only the first 4 items for the preview; "View All" passes cursor for full list
- "View All" navigates to Random Pet Posts screen — uses endpoint `GET /families/{id}/posts?type=random` (Screen 3)

---

## User Flow Diagrams

### Open My Pets Tab

```
User taps My Pets tab
  └─> [not logged in] → redirect to Login
  └─> [logged in, no active family] → empty state: "No active family"
        └─> Tap "Set active family" → Profile Settings
  └─> [logged in, has active family]
        └─> GET /users/me/active-family
              └─> Render family card + pet rows + parents (loaded)
                    └─> GET /families/{id}/random-media?limit=20
                          └─> Render random pets grid
```

### Manage Parents

```
User taps Manage Parents
  └─> Open Parents bottom sheet (data already loaded from AY)
        ├─ Owner: tap Remove on a parent → confirmation → DELETE /families/{id}/parents/{user_id}
        ├─ Owner: tap Cancel on invite → DELETE /families/{id}/invites/{user_id}
        └─ Owner: tap Invite Another Parent → search modal → POST /families/{id}/invites
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No active family | Empty state: family card area shows "No active family" + "Go to Settings" button |
| `random_count = 0` | Hide "randoms" from stats line; RANDOM PETS section shows empty state |
| `viewer_role = parent` (not owner) | Edit button hidden; Parents popup is read-only (no Remove/Cancel/Invite actions) |
| `health_status = check` | Amber `CHECK` badge on pet row |
| Story button | Disabled (greyed out, no action) |
| No pets in family | Pet rows section shows empty state: "No pets yet — create a post to add pets" |
| Random Pets grid — tap cell | Navigate to Post Detail for the parent post |
| `[+]` in Random Pets header | Navigate to Create Post screen |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | "Manage My Pets" row | Removed — pet rows link directly to Pet Detail |
| 2 | Edit button visibility | Owner only; hidden for parent members |
| 3 | Parents popup actions | Owner can Remove/Cancel/Invite; parents see read-only list |
| 4 | Health status detail | To be specced in a separate Pet Detail / Health screen |
| 5 | `[+]` in Random Pets | Navigate to Create Post screen; AI scan in this context = breed-detect only (skip family pet matching — result is `breed` or `random`, never `pet`) |
| 6 | Lock icon on pet avatar | Not used — health status shown via badge only (no lock icon) |
| 7 | MY PET POSTS section | Reuses endpoint R (`GET /families/{id}/posts`) from Screen 3; no new endpoint needed |
