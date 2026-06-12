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
  Right: Search icon | Messages icon ✉ (red dot if any unread chat; opens screen_10) | Notifications icon 🔔 (red dot if any unread activity; opens screen_22) | Profile avatar

[Scrollable area]
  ├── Family Info Card (active family)
  │     ├── Avatar (stacked pet photos, rotates)
  │     ├── Family name
  │     ├── @tag  ·  📍 cityCode, countryCode
  │     ├── N pets · N randoms · N followers
  │     └── [Post button]  [Edit button]
  │
  ├── Pet Rows (one per named pet, no pagination)
  │     ├── [avatar]  Name · Breed  [🔒 if private]
  │     │             sex · age · N posts
  │     │             [health status badge]
  │     │             [Story]  [>]
  │     └── (repeat for each pet)
  │
  ├── Manage Parents row  [>]
  │     └── → opens Parents bottom sheet
  │
  ├── Rescues row  [>]   ← only when active family is a charity
  │     └── → opens Manage Rescues (screen_27)
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
| `avatarUrl` / `petAvatars` | Stacked pet avatars — same rules as Screen 3: 0 public pet = default family avatar, 1 = single pet avatar, ≥2 = stacked rotating (up to 5). Only **public pets** shown in stacked display; family avatar (if explicitly set) takes precedence over stacked pets |
| `name` | Family display name |
| `tag` | `@tag` |
| `cityCode`, `countryCode` | Displayed as `📍 HCM, VN` |
| `petCount` | `N pets` |
| `randomCount` | `N randoms` — hidden if 0 |
| `social.followersCount` | `N followers` |

> followersCount trả số thô; client tự format "3.6k" theo locale (bỏ followerCountDisplay — i18n client-side).

> PR #882: `viewerRole` (FamilyRole) và `isPrimary` (Boolean) nay khả dụng trên family object — có thể dùng cho UI logic quyền hạn thành viên nếu cần.

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
| `avatarUrl` | Pet avatar |
| `isPublic` | 🔒 lock icon shown on row if `isPublic = false` (private pet) |
| `name` | Pet name |
| `breed` | `BreedGQL` — use `breed.nameVi` (Vietnamese) or `breed.nameEn` for display (truncated with `...` if long); `null` if unknown |
| `sex` | `Male` / `Female` / `Unknown` |
| `ageMonths` | Tổng số tháng tuổi (Int). Client render "3 tuổi"/"3 years" theo locale + birthDatePrecision. |
| `postCount` | `N posts` |
| `healthStatus` | Badge — see Health Status below |

> Tuổi trả về dạng `ageMonths` (Int) — client tự format hiển thị theo locale + `birthDatePrecision`, server không format chuỗi.

**Health Status badge** (simplified aggregate shown on row — matches Screen 9 values):

| Status | Badge | Color | Meaning |
|--------|-------|-------|---------|
| `CHECKING` | `CHECKING` | Grey | AI analysis in progress / no data yet |
| `NORMAL` | `NORMAL` | Green | No issues |
| `CONCERN` | `CONCERN` | Amber | Needs attention |
| `BAD` | `BAD` | Orange | Serious issues |
| `CRITICAL` | `CRITICAL` | Red | Urgent — vet visit needed |

> Detailed per-tab health logic (Food / Behavior / Med) defined in Screen 9.

**Story button:** → opens the pet's **Story** (`screen_29`) — the chronological photo/video timeline of that pet, with a Play slideshow. Available to family members (this screen is always viewed by a member).

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
[avatar]  Cecilia Tran  [PARENT]      [✉]  [Remove]
──────────────────────────────────────────────────
[avatar]  Thao Nguyen  [INVITED]           [Cancel]
──────────────────────────────────────────────────
[+]  Invite Another Parent                 [>]
```

**Row types:**

| Row | Badges | Message icon `[✉]` | Action | Who can see action |
|-----|--------|--------------------|--------|--------------------|
| Owner (self) | `YOU` + `OWNER` | Hidden (can't DM yourself) | None | — |
| Accepted parent | `MEMBER` | Shown — opens a DM | Remove | Owner only |
| Pending invite | `INVITED` | Hidden (not a member yet) | Cancel | Owner only |

**Message icon `[✉]` (accepted co-parents only):**
- Tap → start a **DM** with that user: `StartThread (BI)` `{ senderType: USER (self), receiverType: USER }` → open the thread in the Messages screen's Thread View (screen_10). Reuses existing DM thread if one exists.

**Remove action:**
- Confirmation dialog → `RemoveFamilyMember mutation (AS)` (endpoint AS from Screen 6)
- Row removed immediately on confirm

**Cancel invite action:**
- `CancelParentInvite mutation (AR)` (endpoint AR from Screen 6)
- Row removed immediately

**Invite Another Parent:**
- Opens invite search modal (same as Screen 6 → Section: Invite Search Modal)
- Uses endpoint AP (search users) + AQ (send invite) from Screen 6

> This bottom sheet shares the same data model and all API endpoints with Screen 6 (Parents section). No new endpoints needed.

---

### 3a. Rescues Row (charity families only)

```
🐾  Rescues                                 [>]
```

- **Shown only when the active family is a charity** (`family.familyType = CHARITY`). Hidden for all non-charity families.
- Tap → navigate to **Manage Rescues** (`screen_27`) — the charity's Open / Adopted rescue listings, where they post, edit, mark adopted, or reopen.
- Available to all charity members (owner + parents), consistent with the post/close permission (member of the charity while it's active).
- No new endpoint here — Manage Rescues uses its own queries/mutations (`MyRescues CN`, `CloseRescue CP`, `ReopenRescue CQ`, `UpdateRescue CS`, `CreateRescue CO`).

---

### 4. Random Pets Section

Displays posts where at least one media has `mediaTag.type = "random" AND (breed IS NOT NULL OR species IS NOT NULL)` — AI detected an animal but no named pet was matched.

**Preview grid (2×2, max 4 cells):**

Each cell shows:
- Media thumbnail (top)
- `breed` name if available, otherwise `species` name — in bold (below thumbnail)
- `📍 cityCode, countryCode · time` — time follows the same display rules as post cards (see `screen_1_home_explore.md` → Post Card → Time display rules). E.g. `"HCMC, VN · 3h"`, `"HCMC, VN · 2d"`, `"HCMC, VN · 28 May"`

- Tap any cell → **Post Detail screen** for that post
- If total breed-tagged posts > 4: show **"View All →"** link below grid
- **"View All →"** → **Random Pet Posts screen** (same pattern as Pet Posts screen in Screen 3, but filtered to posts with `mediaTag.type = "random" AND breed IS NOT NULL`; header title: "Random Pets")
- If 0 breed-tagged posts → hide section entirely

**`[+]` button:**
- Tap → **Create Post screen** (Screen 7)
- In this context, AI scan **skips the family pet matching step** — result can only be `breed` or `random`, never `pet`. This means the scan API is called without `familyId` (or with a flag `skipPetMatch: true`)

---

### 5. My Pet Posts Section

All posts belonging to the active family, same as Family Posts screen (Screen 3).

- Section header: **"MY PET POSTS"** + list/grid view toggle
- Default: list view — canonical post card (Screen 1); all canonical tap interactions apply: tap **family name** → Family Posts, tap **author name** → User Posts, tap **pet badge** → Pet Posts
- Grid view: same 3-column thumbnail grid as Screen 3
- Sorted `createdAt` desc (newest first)
- 10 posts per page, infinite scroll
- API: `FamilyPosts query (R)` (endpoint R from Screen 3)

---

## API Endpoints Required

> Most data for this screen reuses existing endpoints. New endpoints listed below. All calls go to `POST /graphql`.

### AY. Query: active family for My Pets (`me.primaryFamily` + `petsByFamily`)

There is **no dedicated `activeFamily` query** (rejected by canonical reconcile #877 — petapp-be issue #819 closed). The active family is the user's **primary family**, fetched directly via `me { primaryFamily }` (backed by the `GetUserPrimaryFamily` RPC — one round-trip, no client-side filtering needed). The pet list is a **separate** query `petsByFamily(familyId)` — `Family.pets` is intentionally not a nested field (avoids an unbatched N+1).

So the My Pets screen issues **two queries**:

**Query 1 — primary family header + parents (`me.primaryFamily`):**
```graphql
query MyPetsHeader {
  me {
    primaryFamily {
      id
      name
      tag
      avatarUrl
      petAvatars            # ordered public pet avatars (up to 5)
      petCount
      viewerRole            # FamilyRole: OWNER | ADMIN | MEMBER
      location {            # replaces flat city/cityCode/country/countryCode
        city
        cityCode
        country
        countryCode
      }
      social {
        followersCount
      }
      members {             # replaces `parents[]` — for the Parents bottom sheet
        userId
        displayName
        username
        avatarUrl
        role                # FamilyRole
        status              # MemberStatus: JOINED | INVITED
      }
    }
  }
}
```

**Query 2 — pets of the active family (`petsByFamily`):**
```graphql
query MyPetsList($familyId: ID!) {        # familyId = me.primaryFamily.id from query 1
  petsByFamily(familyId: $familyId) {
    id
    name
    avatarUrl
    breed { nameVi nameEn }
    isPublic
    sex                      # PetSex: MALE | FEMALE | UNKNOWN
    ageMonths                # Int — client formats "3 years" / "5 months" per locale
    postCount                # Int! — số post gắn pet (resolver cross-service, batched DataLoader)
  }
}
```

**Variables:** query 1 `{}`; query 2 `{ "familyId": "<me.primaryFamily.id>" }`

**Response `200 OK` (query 1):**
```json
{
  "data": {
    "me": {
      "primaryFamily": {
        "id": "fam_xyz",
        "name": "Minh's Family",
        "tag": "minhfamily",
        "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
        "petAvatars": ["..."],
        "petCount": 3,
        "viewerRole": "OWNER",
        "location": { "city": "Hồ Chí Minh", "cityCode": "HCM", "country": "Việt Nam", "countryCode": "VN" },
        "social": { "followersCount": 287 },
        "members": [
          { "userId": "user_001", "displayName": "Minh Dang", "username": "minhdang", "avatarUrl": "...", "role": "OWNER", "status": "JOINED" },
          { "userId": "user_002", "displayName": "Cecilia Tran", "username": "ceciliatran", "avatarUrl": "...", "role": "MEMBER", "status": "JOINED" },
          { "userId": "user_003", "displayName": "Thao Nguyen", "username": "thaonguyen", "avatarUrl": "...", "role": "MEMBER", "status": "INVITED" }
        ]
      }
    }
  }
}
```

**Response `200 OK` (query 2):**
```json
{
  "data": {
    "petsByFamily": [
      { "id": "pet_111", "name": "Bụi", "avatarUrl": "...", "breed": { "nameVi": "Mèo vằn cam", "nameEn": "Orange Tabby Cat" }, "isPublic": true, "sex": "MALE", "ageMonths": 36, "postCount": 12 },
      { "id": "pet_222", "name": "Măng", "avatarUrl": "...", "breed": { "nameVi": "Ngựa buckskin", "nameEn": "Buckskin Pony" }, "isPublic": false, "sex": "FEMALE", "ageMonths": 60, "postCount": 3 }
    ]
  }
}
```

**Notes:**
- `me.primaryFamily` is `null` if the user has no primary family → render empty state (replaces the `ACTIVE_FAMILY_NOT_SET` error).
- `viewerRole` (`OWNER` | `ADMIN` | `MEMBER`) controls the Edit button + Parents-popup management actions.
- `members` is included in query 1 to avoid a separate call when opening the Parents bottom sheet.
- `petsByFamily` enforces privacy: members see all pets; non-members see only `isPublic: true`.
- `postCount` (per-pet) is now implemented (petapp-be #901) — `Pet.postCount: Int!`, a cross-service resolver counting posts tagged with the pet (via `post_pets`), batched through a DataLoader to avoid N+1 when listing pets. Used for the `N posts` sub-label.
- Dropped vs the old `activeFamily` shape (counter-canonical): `gender`→`sex`, server-side `ageDisplay`→client-formatted `ageMonths`, `parents`→`members`, `randomCount` (random-media tag is a separate deferred feature — see AZ), and per-pet `healthStatus` (not yet implemented; `healthStatus` would aggregate from the health/AI service — file a separate Pet issue if the screen needs it).

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### AZ. Query: `FamilyRandomMedia`

Fetch breed-tagged media for the Random Pets section preview (2×2 grid + View All list).  
**Auth:** Required

**Operation:**
```graphql
query FamilyRandomMedia($familyId: ID!, $first: Int! = 20, $after: String) {
  familyRandomMedia(familyId: $familyId, first: $first, after: $after) {
    mediaCount
    media {
      edges {
        cursor
        node {
          id
          thumbnailUrl
          postId
          breed
          cityCode
          countryCode
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
{
  "familyId": "fam_xyz",
  "first": 20,
  "after": null
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "familyRandomMedia": {
      "mediaCount": 12,
      "media": {
        "edges": [
          {
            "cursor": "cursor_media_003",
            "node": {
              "id": "media_003",
              "thumbnailUrl": "https://cdn.petapp.com/media/003_thumb.jpg",
              "postId": "post_abc",
              "breed": "British Shorthair",
              "cityCode": "HCM",
              "countryCode": "VN",
              "createdAt": "2026-05-01T10:00:00Z"
            }
          }
        ],
        "pageInfo": {
          "hasNextPage": true,
          "endCursor": "cursor_media_003"
        }
      }
    }
  }
}
```

> **Note:** `mediaCount` (total media count for the family) is a sibling field on `familyRandomMedia`, not inside the connection — per ADR-0023 `totalCount` does not live in the Relay envelope.

**Notes:**
- `thumbnailUrl` used directly in the 2×2 grid cells
- `breed`, `cityCode`, `countryCode`, `createdAt` used for cell subtitle display: `breed name · 📍 HCM, VN · 1 mo`
- Screen fetches only the first 4 items for the preview; "View All" passes cursor for full list
- "View All" navigates to Random Pet Posts screen — uses Query `U. RandomPetPosts` (screen_3)

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `FAMILY_NOT_FOUND` | Family does not exist |

---

## User Flow Diagrams

### Open My Pets Tab

```
User taps My Pets tab
  └─> [not logged in] → redirect to Login
  └─> [logged in, me.primaryFamily == null] → empty state: "No active family"
        └─> Tap "Set active family" → Profile Settings
  └─> [logged in, me.primaryFamily != null]
        └─> Query 1: me { primaryFamily } (AY) → then Query 2: petsByFamily(familyId) (AY)
              └─> Render family card + pet rows + parents/members (loaded)
                    └─> FamilyRandomMedia query (AZ)
                          └─> Render random pets grid
```

### Manage Parents

```
User taps Manage Parents
  └─> Open Parents bottom sheet (data already loaded from AY)
        ├─ Owner: tap Remove on a parent → confirmation → RemoveFamilyMember mutation (AS)
        ├─ Owner: tap Cancel on invite → CancelParentInvite mutation (AR)
        └─ Owner: tap Invite Another Parent → search modal → InviteParent mutation (AQ)
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No active family | Empty state: family card area shows "No active family" + "Go to Settings" button |
| `randomCount = 0` | Hide "randoms" from stats line; RANDOM PETS section shows empty state |
| `viewerRole = parent` (not owner) | Edit button hidden; Parents popup is read-only (no Remove/Cancel/Invite actions) |
| `healthStatus = CONCERN` | Amber `CONCERN` badge on pet row |
| Story button | Opens the pet's Story (`screen_29`) |
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
| 7 | MY PET POSTS section | Reuses `FamilyPosts query (R)` from Screen 3; no new endpoint needed |
