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
  │     ├── Section header: "RANDOM PETS"  [+]
  │     └── 2-column media grid (thumbnails of breed-tagged media)
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

All named pets of the active family. No pagination — all shown at once.

Each row:

| Field | Display |
|-------|---------|
| `avatar_url` | Pet avatar. If pet has a pending health check, show a 🔒 lock icon overlay on avatar. |
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
| `check` | `CHECK` | Amber/Orange | Needs attention — TBD (health detail to be specced later) |

> Full health status logic will be defined in a separate spec. For now, display only `NORMAL` and `CHECK`.

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

Displays media where `media_tag.type = "breed"` — AI detected a breed but no named pet was matched.

- Section header: **"RANDOM PETS"** + `[+]` button (top-right)
- **`[+]` button:** tap → navigate to **Create Post screen** (same as Post button)
- **Grid:** 2-column thumbnail grid, newest first
- Each cell: thumbnail of the media item
- Tap cell → navigate to **Post Detail screen** for the post containing that media
- Paginated: load 20 thumbnails at a time, load more on scroll

---

## API Endpoints Required

> Most data for this screen reuses existing endpoints. New endpoints listed below.

### AY. `GET /users/me/active-family`

Fetch the active family with full detail for the My Pets screen.  
**Auth:** Required

**Response `200 OK`:**
```json
{
  "id": "fam_xyz",
  "name": "Minh's Family",
  "tag": "minhfamily",
  "city": "Hồ Chí Minh",
  "city_code": "HCM",
  "country": "Việt Nam",
  "country_code": "VN",
  "avatar_url": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
  "pet_avatars": ["..."],
  "pet_count": 3,
  "random_count": 10,
  "follower_count": 287,
  "viewer_role": "owner",
  "pets": [
    {
      "id": "pet_111",
      "name": "Bụi",
      "avatar_url": "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
      "breed": "Orange Tabby Cat",
      "gender": "male",
      "age_display": "3 years",
      "post_count": 47,
      "health_status": "normal"
    },
    {
      "id": "pet_222",
      "name": "Măng",
      "avatar_url": "https://cdn.petapp.com/pets/pet_222/avatar.jpg",
      "breed": "Buckskin Pony",
      "gender": "female",
      "age_display": "5 years",
      "post_count": 24,
      "health_status": "check"
    }
  ],
  "parents": [
    { "id": "user_001", "name": "Minh Dang", "tag": "minhdang", "avatar_url": "...", "role": "owner", "status": "joined" },
    { "id": "user_002", "name": "Cecilia Tran", "tag": "ceciliatran", "avatar_url": "...", "role": "parent", "status": "joined" },
    { "id": "user_003", "name": "Thao Nguyen", "tag": "thaonguyen", "avatar_url": "...", "role": "parent", "status": "invited" }
  ]
}
```

**Notes:**
- `viewer_role`: `"owner"` | `"parent"` — controls visibility of Edit button and management actions in Parents popup
- `parents` included here to avoid a separate API call when opening the bottom sheet

---

### AZ. `GET /families/{family_id}/random-media`

Fetch paginated random pet media (breed-tagged) for the Random Pets section.  
**Auth:** Required

**Query Parameters:** `cursor`, `limit=20`

**Response `200 OK`:**
```json
{
  "media": [
    {
      "id": "media_003",
      "url": "https://cdn.petapp.com/media/003.jpg",
      "thumbnail_url": null,
      "post_id": "post_abc",
      "media_tag": {
        "type": "breed",
        "breed": "British Shorthair",
        "color": "grey",
        "pet": null
      }
    }
  ],
  "next_cursor": "...",
  "has_more": true
}
```

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
| `health_status = check` | Amber badge on pet row + 🔒 lock icon on pet avatar |
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
| 5 | `[+]` in Random Pets | Navigate to Create Post screen |
