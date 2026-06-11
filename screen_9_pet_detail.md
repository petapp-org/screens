# Screen 9: Pet Detail

## Overview

Full detail view for a single named pet within a family.  
Navigated to from: tapping a pet row in **My Pets (Screen 8)** only.  
Accessible to: family members (owner + parents) — My Pets is always viewed by a family member, so no auth check needed on entry.

---

## UI Layout

```
[Header — pet switcher]
  [pet 1 avatar ← active, highlighted]  [pet 2 avatar]  [pet 3 avatar]  [+]
  (horizontal scroll if many pets)

[Pet identity row]
  Pet Name — Breed                                          [▶ Story]  [...]
  sex · age · N posts

[Missing banner — only when status = missing]
  ⚠️ MISSING · Last seen HCM · 2 days ago        [Mark as Found]

[Category Tabs]
  Health  |  Food  |  Behavior  |  Med/Vac

[Tab content area — HTML blob rendered as-is]
  (content switches with the active tab)

[Below the tabs — always visible, independent of the active tab]
  [📍 Report Missing button]   (all family members)
  [Delete Pet button]          (owner only)
```

---

## Components

### 1. Pet Switcher (Header)

Shows all **non-deleted** pets belonging to the same family, as circular avatar chips.

- Active pet: highlighted with colored border ring
- Tap another pet avatar → switch to that pet's detail (same screen, data reloads)
- Horizontal scroll if family has many pets
- **`[+]` button** (rightmost, dashed circle): tap → navigate to **Create Post screen** (Screen 7)
  - Same behavior as `[+]` in Random Pets section (Screen 8): AI scan in Create Post will match against family pets as normal
  - After publish → navigate to **My Pets tab** (Screen 8)
- Deleted pets (`isDeleted = true`) are excluded from the switcher

---

### 2. Pet Identity Row

```
Bụi — Orange Tabby Cat                                   [▶ Story]  [...]
Male · 3 years · 47 posts
```

| Field | Display |
|-------|---------|
| `name` | Pet name |
| `species` | `SpeciesGQL` — nested object; use `species.name` for display (e.g. `"Cat"`) and `species.iconEmoji` for icon |
| `breed` | `BreedGQL` — nested object; use `breed.nameVi` for Vietnamese display (e.g. `"Mèo lông ngắn Anh"`), `breed.nameEn` for English — `null` if unknown |
| `isPublic` | `true` / `false` — shown as a 🔒 lock badge next to pet name when `false` |
| `sex` | `Male` / `Female` / `Unknown` |
| `ageMonths` | Tổng số tháng tuổi (Int). Client render "3 tuổi"/"3 years" theo locale + birthDatePrecision. |
| `postCount` | Total posts linked to this pet |

> Tuổi trả về dạng `ageMonths` (Int) — client tự format hiển thị theo locale + `birthDatePrecision`, server không format chuỗi.

**`[▶ Story]` button:**
- Opens the pet's **Story** (`screen_29`) — the chronological photo/video timeline + Play slideshow.
- Shown to all family members (Pet Detail is member-only; Story is too).

**`[...]` action menu:**
- "Edit Pet" → opens Edit Pet bottom sheet (see Section 7)
- Owner only; hidden for non-owner parents

---

### 3. Missing Banner

Shown only when `pet.missingStatus != null`.

```
⚠️  MISSING · Last seen HCM · 2 days ago              [Mark as Found]
```

| Field | Display |
|-------|---------|
| `missingStatus.lastSeenAt` | Time string following the same display rules as post cards (see `screen_1_home_explore.md` → Post Card → Time display rules) — how long since the pet was last seen. E.g. `"2d"`, `"28 May"` |
| `missingStatus.lastSeenLocation` | `cityCode` display, e.g. `"HCM"` |

**"Mark as Found" button:**
- Shown to all family members (owner + parents)
- **Disabled** for parents — only owner can tap it
- Tap (owner) → confirmation: *"Mark Bụi as found?"* → `MarkPetFound mutation (BF)`
- Banner removed immediately on success
- Button re-enables automatically when a new missing report is filed (`pet.missingStatus != null`)

---

### 4. Category Tabs

4 fixed tabs. Each tab renders a **pre-generated HTML blob** returned by the API.

| Tab | Data source | Scope |
|-----|-------------|-------|
| Health | AI-generated per post, latest post wins | Pet-level |
| Food | AI-generated per breed, shared | Breed-level |
| Behavior | AI-generated per breed, shared | Breed-level |
| Med/Vac | AI-generated per breed, shared | Breed-level |

**Rendering:**
- HTML blob is rendered directly in a web view / rich text renderer
- Client does not parse or transform the HTML
- Scroll is part of the main screen scroll (not a nested scroll)

---

### 4a. Health Tab

**Content:** HTML blob from latest post's health analysis.

> **Triggers notification:** when the server-side AI analysis (run after a post is published, see screen_7 → `CreatePost`) detects a possible health concern, it fires a `HEALTH_SIGNAL` notification to the pet's family/owner (see screen_22 — Notifications screen). This is a server-side async event, not a client call.

**Status states:**

| Status | Display | Color | Meaning |
|--------|---------|-------|---------|
| `checking` | Spinner + *"Analyzing latest post…"* | Blue | AI processing in queue; no result yet |
| `no_data` | *"No health data detected"* | Grey | AI processed but could not extract health info from media |
| `good` | `GOOD` badge | Green | All health indicators look excellent |
| `normal` | `NORMAL` badge | Green (light) | Healthy baseline, nothing concerning |
| `concern` | `CONCERN` badge | Amber | Some indicators warrant attention |
| `bad` | `BAD` badge | Orange | Multiple concerning indicators; recommend vet visit |
| `critical` | `CRITICAL` badge | Red | Urgent signs detected; see a vet immediately |

When status is `checking` or `no_data`: show status message only, no HTML blob.  
When status is a result state: render HTML blob below the status badge.

**Health content includes** (AI-generated, inside HTML blob):
- Overall assessment summary
- Per-indicator breakdown: Posture, Coat, Body Condition, Ear Position, Tail Carriage, Eye Clarity, etc.
- Each indicator: description + status + photo count from analyzed post
- Source: media from the latest post linked to this pet

---

### 4b. Food Tab

**Content:** HTML blob with breed-level food recommendations (falls back to species-level content if `breed` is null).

Includes:
- Top 5 recommended foods (with brief reason)
- Top 5 toxic / dangerous foods (with warning)

Status states (same logic as Health):

| Status | Display |
|--------|---------|
| `checking` | *"Generating food guide…"* |
| `no_data` | *"No food data available for this breed"* |
| *(populated)* | HTML blob rendered |

---

### 4c. Behavior Tab

**Content:** HTML blob with breed-level behavior guide (falls back to species-level content if `breed` is null).

Includes:
- 5 common body language descriptions + what they mean
- Top 5 things this breed likes
- Top 5 things this breed dislikes

Status states: same pattern as Food tab.

---

### 4d. Med/Vac Tab

**Content:** HTML blob with breed-level medical guide (falls back to species-level content if `breed` is null).

Includes:
- Vaccine schedule by life stage (kitten/puppy, adult, senior)
- Top 5 common diseases for this breed + how to handle / treat

Status states: same pattern as Food tab.

---

### 5. Report Missing Button

```
[📍 Report Missing]   ← below the category tabs, above Delete Pet
```

- Positioned **below the category tabs** — always visible regardless of which tab (Health / Food / Behavior / Med-Vac) is active. It is **not** part of any tab's content (the tabs only switch the HTML blob above).
- Tap → opens the **Report Missing form** (see Section 8). The pet is already in context (the pet currently shown), so **no pet selector** is shown.
- Shown to **all family members** (owner + parents).
- **Hidden** when the pet is already missing (`missingStatus != null`) — in that state the Missing banner + "Mark as Found" (Section 3) take over.

---

### 6. Delete Pet Button

```
[Delete Bụi]   ← red/destructive color, below the Report Missing button
```

- Owner only; hidden for non-owner parents
- Tap → confirmation dialog:
  *"Delete Bụi? This pet will be removed from your family's pet list. Posts linked to Bụi will remain. This action can be undone."*
  → Confirm → `DeletePet mutation (BD)` (soft delete: `isDeleted = true`)
- Pet immediately disappears from:
  - Pet switcher (header)
  - My Pets pet rows (Screen 8)
  - Family Posts pet list (Screen 3)
  - Any filter or picker referencing pets
- Posts and media tags are **not modified** — historical posts retain the pet badge (pet still exists in DB, just hidden from active UI)
- Navigate back to My Pets tab after delete

---

### 7. Edit Pet Bottom Sheet

Opened from `[...]` → "Edit Pet". Owner only.

| Field | Required | Notes |
|-------|----------|-------|
| Avatar | No | Replace pet avatar — upload via `SignUploadBatch (BV)` `{ items: [{ purpose: "AVATAR", ... }] }` (screen_4) → use `list[0].publicUrl`; no AI scan |
| Name | Yes | Pet display name |
| Public | Yes | Toggle `isPublic` on/off; default `true`; when off → pet hidden from Family Posts, not searchable, non-member post card badge treated as random |
| Species | Yes | Read-only if set by AI scan; editable if entered manually |
| Breed | No | Read-only if set by AI scan; editable if entered manually; can be left blank |
| Gender | Yes | `male` / `female` / `unknown` |
| Birthday | No | Date picker |
| Weight | No | Number + unit (kg) |

- Submit → `UpdatePet mutation (BC)`
- `species` and `breed` fields: if set by AI scan, shown as read-only with label *"Set by AI"*; owner can override by unlocking (tap lock icon → confirmation)

---

### 8. Report Missing Form

A **full-screen form** (`← Report Missing`). Shared by two entry points; the only difference is whether a **pet selector** appears at the top.

**Entry modes:**

| Opened from | Pet selector | Pet scope |
|-------------|--------------|-----------|
| **Pet Detail** — Report Missing button (Section 5) | Not shown — pet is the one on screen | — |
| **Lost Pets screen** banner (`screen_18`) | **Shown** at top — pick the pet | The user's **active family** pets, **including private** (`isPublic = false`); already-missing / deleted excluded. Reporting ignores pet privacy. See `screen_18` for selector details. |

```
[Header]  ← Report Missing

( Pet selector — only from Lost Pets )
  Which pet is missing?   [ Bụi ▼ ]

PHOTOS *                                      (at least 1)
  [ + ]  [ photo①  ★cover ]  [ photo② ]  [ photo③ ]  →  scroll
  · first photo = cover; tap a photo → "Set as cover"
  · reorderable; 1–10; uploaded only, no AI scan

LAST SEEN
  📍 Location *   [ map — drag pin 📍 ]
                  → lat/lng captured; city/country auto-filled below
                  Đà Lạt, Việt Nam
  🕑 When *       [ date + time picker ]   default: now

DESCRIPTION *
  [ multiline — identifying details, collar, behavior, responds to name… ]

[ Submit Report ]   ← full-width; disabled until ALL required fields set
```

| Field | Required | Notes |
|-------|----------|-------|
| `pet` | Yes | Pre-selected (Pet Detail) or via selector (Lost Pets — active family, incl. private) |
| `photos` | **Yes** | **At least 1**, up to 10. Uploaded files only; **ordered**, `photos[0]` = **cover**; reorderable; "Set as cover" moves a photo to the front; no AI scan. On the Lost Pet Detail screen (`screen_19`) these render as a **single header carousel** (cover first) — there is no separate "more photos" block (decision: merged). |
| `lastSeenLocation` | Yes | User drags a pin on the map → `{ lat, lng }`; reverse-geocoded → `{ city, cityCode, country, countryCode }` shown for confirmation. **No district.** |
| `lastSeenAt` | Yes | Date + time the pet was **last seen** (distinct from `reportedAt`); defaults to now; cannot be in the future. |
| `description` | **Yes** | Free text — identifying details to help others recognise the pet. Cannot be empty. |

> **Submit** is enabled only when **all required fields** are set: a pet, ≥ 1 photo, a map location, a last-seen time, and a non-empty description.

**Upload:** each photo uploaded via `SignUploadBatch (BV)` `{ items: [{ purpose: "LOST_PET", ... }] }` (screen_4) → use `list[0].publicUrl`; no AI scan (missing photos are never scanned).

**On submit → `ReportMissing mutation (BE)`:**
- `pet.missingStatus` set with the full shape: `lastSeenLocation` (+ `lat`/`lng`), `lastSeenAt`, `description`, ordered `photos` (cover first), `reportedBy` (the caller), `reportedAt`.
- `MISSING` badge appears across the UI; Missing banner appears on Pet Detail (Section 3).
- Push notification to family **followers**: *"Bụi from Minh's Family is missing! Last seen in Đà Lạt"*.
- The report appears in **Lost Pets** (`screen_17` / `screen_18`) for the report's city, and on the **Lost Pet Detail** screen (`screen_19`).

---

## AI Processing Pipeline (background)

Triggered after a post is published that links to a pet (new or existing):

```
Post published → mediaTag.type = pet → pet identified/created
  │
  ├─ Health analysis (per post)  [server-side async process — not a client API call]
  │     └─> pet.healthStatus = "checking" while in queue
  │           └─> Complete → update pet health record + status
  │           └─> No result → pet.healthStatus = "no_data"
  │
  └─ Breed data (if breed not yet generated)  [server-side async process — not a client API call]
        └─> All 3 tabs (Food, Behavior, Med/Vax) generated in parallel
              └─> Stored as HTML blobs on breed record
              └─> pet tab statuses update from "checking" → populated
```

**Status during processing:**
- `checking` is set immediately when the pet is linked to a new post
- Individual tabs update independently as each generation completes
- Health tab can be `checking` while Food tab is already populated (or vice versa)

---

## API Endpoints Required

All calls go to `POST /graphql`.

### BA. Query: `Pet`

Fetch full pet detail including all tab content.
**Auth:** Required (family member only)

**Operation:**
```graphql
query Pet($petId: ID!) {
  pet(petId: $petId) {
    id
    name
    species {
      name
      iconEmoji
    }
    breed {
      nameVi
      nameEn
    }
    breedId
    isPublic
    sex
    ageMonths
    birthDate
    weightKg
    avatarUrl
    postCount
    healthStatus
    missingStatus {
      id
      reportedAt
      reportedBy {
        id
        displayName
        avatarUrl
      }
      lastSeenLocation {
        city
        cityShortName
        cityCode
        country
        countryCode
        lat
        lng
      }
      lastSeenAt
      description
      photos
    }
    isDeleted
    familyId
    viewerRole
    tabs {
      health {
        status
        html
        sourcePostId
        analyzedAt
      }
      food {
        status
        html
        sourcePostId
        analyzedAt
      }
      behavior {
        status
        html
        sourcePostId
        analyzedAt
      }
      medVax {
        status
        html
        sourcePostId
        analyzedAt
      }
    }
  }
}
```

**Variables:**
```json
{ "petId": "pet_111" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "pet": {
      "id": "pet_111",
      "name": "Bụi",
      "species": { "name": "Cat", "iconEmoji": "🐱" },
      "breed": { "nameVi": "Mèo vằn cam", "nameEn": "Orange Tabby Cat" },
      "breedId": "breed_orange_tabby_cat",
      "isPublic": true,
      "sex": "MALE",
      "ageMonths": 36,
      "birthDate": "2023-01-15",
      "weightKg": 4.2,
      "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
      "postCount": 47,
      "healthStatus": "NORMAL",
      "missingStatus": null,
      "isDeleted": false,
      "familyId": "fam_xyz",
      "viewerRole": "owner",
      "tabs": {
        "health": {
          "status": "NORMAL",
          "html": "<div>...</div>",
          "sourcePostId": "post_abc",
          "analyzedAt": "2026-06-01T10:00:00Z"
        },
        "food": {
          "status": "NORMAL",
          "html": "<div>...</div>",
          "sourcePostId": null,
          "analyzedAt": null
        },
        "behavior": {
          "status": "NORMAL",
          "html": "<div>...</div>",
          "sourcePostId": null,
          "analyzedAt": null
        },
        "medVax": {
          "status": "NORMAL",
          "html": "<div>...</div>",
          "sourcePostId": null,
          "analyzedAt": null
        }
      }
    }
  }
}
```

**Tab status values:** `CHECKING` | `NO_DATA` | `GOOD` | `NORMAL` | `CONCERN` | `BAD` | `CRITICAL`

**Notes:**
- When `tabs.health.status = CHECKING` or `NO_DATA`: `html` is `null`
- Same for Food/Behavior/Med/Vac tabs
- `viewerRole`: `"owner"` | `"parent"` — controls visibility of Edit / Delete / Mark as Found actions
- `missingStatus` is `null` when the pet is not missing. When set: `photos` is an **ordered** list with `photos[0]` = cover; `reportedBy` is the family member who filed the report; `lastSeenAt` is when the pet was last seen (distinct from `reportedAt`). These power the Missing banner (Section 3) and the Lost Pet Detail screen (`screen_19`).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_FAMILY_MEMBER` | Caller is not a member of this pet's family |
| `404` | `PET_NOT_FOUND` | Pet does not exist or is soft-deleted |

---

### BB. Query: `PetsByFamily`

Fetch all active (non-deleted) pets for the family — used to populate the pet switcher.
**Auth:** Required (family member)

**Operation:**
```graphql
query PetsByFamily($familyId: ID!) {
  petsByFamily(familyId: $familyId) {
    id
    name
    avatarUrl
    isPublic
    healthStatus
    missingStatus {
      reportedAt
      lastSeenLocation {
        city
        cityCode
        country
        countryCode
      }
    }
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
    "petsByFamily": [
      {
        "id": "pet_111",
        "name": "Bụi",
        "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg",
        "isPublic": true,
        "healthStatus": "NORMAL",
        "missingStatus": null
      }
    ]
  }
}
```

---

### BC. Mutation: `UpdatePet`

Edit pet info.
**Auth:** Required (owner only)

**Operation:**
```graphql
mutation UpdatePet($petId: ID!, $input: UpdatePetInput!) {
  updatePet(petId: $petId, input: $input) {
    id
    name
    species {
      name
      iconEmoji
    }
    breed {
      nameVi
      nameEn
    }
    breedId
    isPublic
    sex
    ageMonths
    birthDate
    weightKg
    avatarUrl
    postCount
    healthStatus
    missingStatus {
      reportedAt
      lastSeenLocation {
        city
        cityCode
        country
        countryCode
      }
    }
    isDeleted
    familyId
  }
}
```

**Variables:**
```json
{
  "petId": "pet_111",
  "input": {
    "name": "Bụi Bụi",
    "species": "Cat",
    "isPublic": true,
    "sex": "MALE",
    "birthDate": "2023-01-15",
    "weightKg": 4.5,
    "avatarUrl": "https://cdn.petapp.com/media/new_avatar.jpg",
    "breed": "Orange Tabby Cat",
    "forceBreedUpdate": false
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "updatePet": {
      "id": "pet_111",
      "name": "Bụi Bụi",
      "species": { "name": "Cat", "iconEmoji": "🐱" },
      "breed": { "nameVi": "Mèo vằn cam", "nameEn": "Orange Tabby Cat" },
      "breedId": "breed_orange_tabby_cat",
      "isPublic": true,
      "sex": "MALE",
      "ageMonths": 36,
      "birthDate": "2023-01-15",
      "weightKg": 4.5,
      "avatarUrl": "https://cdn.petapp.com/media/new_avatar.jpg",
      "postCount": 47,
      "healthStatus": "NORMAL",
      "missingStatus": null,
      "isDeleted": false,
      "familyId": "fam_xyz"
    }
  }
}
```

**Notes:**
- `breed` field: if was set by AI scan (`breedSource = "AI"`), override requires explicit `forceBreedUpdate: true` in input
- Changing `breed` triggers re-generation of Food/Behavior/Med/Vax tabs (queued)

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_OWNER` | Caller is a parent, not the owner |
| `403` | `NOT_FAMILY_MEMBER` | Caller is not a member of this pet's family |
| `404` | `PET_NOT_FOUND` | Pet does not exist or is soft-deleted |
| `409` | `BREED_AI_LOCKED` | Attempt to change AI-set breed without `forceBreedUpdate: true` |
| `422` | `INVALID_SPECIES` | Provided species value does not exist in the DB |
| `422` | `INVALID_BREED` | Provided breed does not belong to the given species |

---

### BD. Mutation: `DeletePet`

Soft-delete a pet.
**Auth:** Required (owner only)

**Operation:**
```graphql
mutation DeletePet($petId: ID!) {
  deletePet(petId: $petId)
}
```

**Variables:**
```json
{ "petId": "pet_111" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "deletePet": true
  }
}
```

**Behaviour:**
- Sets `isDeleted = true` on the pet record
- Pet excluded from all active listings and pickers
- Posts and `mediaTag` references are not modified
- Pet record remains in DB (restorable)

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_OWNER` | Caller is not the family owner |

---

### BE. Mutation: `ReportMissing`

Report a pet as missing.
**Auth:** Required (family member)

> **Triggers notifications** (see screen_22 — Notifications screen): a `PET_MISSING` notification to the family's members/followers, and a `MISSING_NEARBY` notification to other users near the last-seen location.

**Operation:**
```graphql
mutation ReportMissing($petId: ID!, $input: MissingReportInput!) {
  reportMissing(petId: $petId, input: $input) {
    id
    petId
    reportedAt
    reportedBy {
      id
      displayName
      avatarUrl
    }
    lastSeenLocation {
      city
      cityShortName
      cityCode
      country
      countryCode
      lat
      lng
    }
    lastSeenAt
    description
    photos
  }
}
```

**`MissingReportInput`:**
```json
{
  "lastSeenLocation": {
    "city": "string", "cityCode": "string",
    "country": "string", "countryCode": "string",
    "lat": "float", "lng": "float"
  },
  "lastSeenAt": "ISO 8601 datetime (not in the future)",
  "description": "string (required, non-empty)",
  "photos": ["string url, 1–10, ordered — photos[0] = cover"]
}
```

- `lat`/`lng` come from the map pin; `city`/`cityCode`/`country`/`countryCode` are reverse-geocoded (client may send what it resolved, server may re-validate). The server derives **`cityShortName`** (curated short label) — the client need not send it; it is returned on read for display on Lost Pets rows / detail.
- `photos` are pre-uploaded `publicUrl`s from `SignUploadBatch (BV)` `{ items: [{ purpose: "LOST_PET", ... }] }` (`list[0].publicUrl`); **at least 1 required**; order is preserved; index 0 is the cover.
- `description` is **required** and must be non-empty.

**Variables:**
```json
{
  "petId": "pet_201",
  "input": {
    "lastSeenLocation": {
      "city": "Đà Lạt",
      "cityCode": "DL",
      "country": "Việt Nam",
      "countryCode": "VN",
      "lat": 11.9404,
      "lng": 108.4583
    },
    "lastSeenAt": "2026-06-07T17:30:00Z",
    "description": "Buckskin Vietnamese pony, dark mane, wearing a green halter rope. Responds to \"Măng\".",
    "photos": [
      "https://cdn.petapp.com/media/miss_cover.jpg",
      "https://cdn.petapp.com/media/miss_2.jpg",
      "https://cdn.petapp.com/media/miss_3.jpg"
    ]
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "reportMissing": {
      "id": "missing_report_001",
      "petId": "pet_201",
      "reportedAt": "2026-06-08T03:00:00Z",
      "reportedBy": {
        "id": "user_001",
        "displayName": "Minh Tuan",
        "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
      },
      "lastSeenLocation": {
        "city": "Đà Lạt",
        "cityShortName": "Đà Lạt",
        "cityCode": "DL",
        "country": "Việt Nam",
        "countryCode": "VN",
        "lat": 11.9404,
        "lng": 108.4583
      },
      "lastSeenAt": "2026-06-07T17:30:00Z",
      "description": "Buckskin Vietnamese pony, dark mane, wearing a green halter rope. Responds to \"Măng\".",
      "photos": [
        "https://cdn.petapp.com/media/miss_cover.jpg",
        "https://cdn.petapp.com/media/miss_2.jpg",
        "https://cdn.petapp.com/media/miss_3.jpg"
      ]
    }
  }
}
```

**Side effects:**
- `pet.missingStatus` set with the full shape above; `reportedBy` = the caller.
- Push notification dispatched to family followers.
- Report becomes visible in Lost Pets (`screen_17` / `screen_18`) for `lastSeenLocation.cityCode` and on Lost Pet Detail (`screen_19`).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_FAMILY_MEMBER` | Caller is not a member of the pet's family |
| `404` | `PET_NOT_FOUND` | Pet does not exist or is soft-deleted |
| `409` | `ALREADY_MISSING` | Pet already has an active missing report |
| `422` | `LAST_SEEN_IN_FUTURE` | `lastSeenAt` is in the future |
| `422` | `MISSING_LOCATION` | `lastSeenLocation` (lat/lng) not provided |
| `422` | `MISSING_PHOTOS` | No photo provided (at least 1 required) |
| `422` | `MISSING_DESCRIPTION` | `description` is empty |

---

### BF. Mutation: `MarkPetFound`

Mark a missing pet as found.
**Auth:** Required (owner only)

> **Triggers notification:** fires a `PET_FOUND` notification (see screen_22 — Notifications screen) to the family's members/followers and to users who were notified of the original `MISSING_NEARBY`.

**Operation:**
```graphql
mutation MarkPetFound($petId: ID!) {
  markPetFound(petId: $petId) {
    missingStatus
  }
}
```

**Variables:**
```json
{ "petId": "pet_111" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "markPetFound": {
      "missingStatus": null
    }
  }
}
```

**Side effects:**
- `pet.missingStatus` cleared
- The active missing **report** is **not deleted** — its `status` flips `MISSING → FOUND`. It leaves all in-app Lost Pets listings (`screen_17` / `screen_18`), but its shared **Lost Pet Detail** link (`screen_19`) keeps resolving and renders the **found state**.
- Optional push notification: *"Good news! Bụi from Minh's Family has been found 🎉"*

---

## User Flow Diagrams

### Open Pet Detail

```
User taps pet row in My Pets
  └─> Pet query (BA) + PetsByFamily query (BB)
        └─> Render pet identity + missing banner (if applicable)
              └─> Default tab: Health
                    ├─ status=checking → show spinner
                    ├─ status=no_data  → show "No data" message
                    └─ status=result   → render HTML blob
```

### Switch Pet

```
User taps another pet avatar in switcher
  └─> Pet query (BA) { id: otherPetId }
        └─> Re-render all sections with new pet data
            (stay on same tab)
```

### Report Missing

```
User taps [Report Missing]  (Pet Detail → pet already in context)
  └─> Report Missing form (full screen, Section 8)
        ├─ [from Lost Pets banner] pick pet first (active family, incl. private)
        └─> drag map pin (location *) + pick When * + photos (optional, set cover) + description
              └─> upload photos (SignUploadBatch, LOST_PET)
                    └─> ReportMissing mutation (BE)
                          └─> success → missing banner appears; followers notified;
                              report visible in Lost Pets + Lost Pet Detail
```

### Delete Pet

```
User taps [Delete Bụi]
  └─> Confirmation dialog
        ├─ Cancel → dismiss
        └─ Confirm → DeletePet mutation (BD)
              └─> success → navigate to My Pets tab
                        pet removed from all active UI
                        posts unchanged
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Pet has no posts yet | All tabs show `checking` until first post is published and AI processes |
| AI processed but no health extracted | `health.status = no_data`; Health tab shows "No health data detected" |
| Breed data already exists for this breed | Food/Behavior/Med/Vac generated from cache; no AI call needed |
| Owner changes breed via Edit Pet | Queues re-generation of all 3 breed tabs; statuses reset to `checking` |
| Pet is `missing` | Missing banner shown; `MISSING` badge shown in pet switcher and My Pets row |
| Non-owner parent | Edit / Delete / Mark as Found actions hidden; can still Report Missing |
| Soft-deleted pet accessed via direct link | `404 PET_NOT_FOUND` |
| Media on posts tagged to deleted pet | Unchanged — pet badge still renders on historical posts using pet data from DB |
| Health tab `sourcePostId` | Tapping the source post thumbnail → navigate to Post Detail |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Delete type | Soft delete (`isDeleted = true`); pet hidden from active UI; posts and tags unchanged |
| 2 | Media tags on deleted pet | Not modified — historical posts retain pet badge (pet still exists in DB) |
| 3 | Tab content format | All tabs render pre-generated HTML blob; client does not parse |
| 4 | Health data scope | Per-post; only latest post's analysis is shown in Health tab |
| 5 | Food/Behavior/Med/Vac scope | Per-breed; shared across all pets of same breed |
| 6 | Tab status independence | Each tab has its own status; tabs update independently as AI completes each task |
| 7 | Report Missing access | All family members (owner + parents) can report missing |
| 8 | Mark as Found access | Owner only |
| 9 | Missing push notification | Sent to family followers on report; optional "found" notification on resolution |
| 10 | `[+]` in pet switcher | Creates post (pet created via AI scan); same as normal Create Post flow |
| 11 | Report Missing placement | A button **below the category tabs** (Section 5), not inside the Health tab; tabs only switch the HTML blob |
| 12 | Report Missing surface | **Full-screen form** (Section 8), shared between Pet Detail and the Lost Pets banner |
| 13 | Report pet selector scope (from Lost Pets) | **Active family** pets only, **including private** (reporting ignores pet privacy) |
| 14 | Missing photos | Single **ordered** photo set; `photos[0]` = cover; rendered as one header carousel on Lost Pet Detail (no separate "more photos") |
| 15 | Last seen | Map pin `lat`/`lng` + auto city/country (no district) **plus** an explicit `lastSeenAt` time, distinct from `reportedAt` |
| 16 | Reporter identity | `missingStatus.reportedBy` recorded → Lost Pet Detail shows "Reported missing from {member / you}" to family members |
