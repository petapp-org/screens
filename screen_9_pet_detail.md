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
  Pet Name — Breed                                                    [...]
  gender · age · N posts

[Missing banner — only when status = missing]
  ⚠️ MISSING · Last seen HCM · 2 days ago        [Mark as Found]

[Category Tabs]
  Health  |  Food  |  Behavior  |  Med/Vac

[Tab content area — HTML blob rendered as-is]
  └─ Health tab also shows:
       [Report Missing button]
  └─ All tabs bottom:
       [Delete Pet button]
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
- Deleted pets (`is_deleted = true`) are excluded from the switcher

---

### 2. Pet Identity Row

```
Bụi — Orange Tabby Cat                                              [...]
Male · 3 years · 47 posts
```

| Field | Display |
|-------|---------|
| `name` | Pet name |
| `species` | Species e.g. `"cat"`, `"dog"`, `"bird"` |
| `breed` | Breed name, e.g. `"British Shorthair"` — `null` if unknown |
| `is_public` | `true` / `false` — shown as a 🔒 lock badge next to pet name when `false` |
| `gender` | `Male` / `Female` / `Unknown` |
| `age_display` | e.g. `"3 years"`, `"5 months"` |
| `post_count` | Total posts linked to this pet |

**`[...]` action menu:**
- "Edit Pet" → opens Edit Pet bottom sheet (see Section 6)
- Owner only; hidden for non-owner parents

---

### 3. Missing Banner

Shown only when `pet.missing_status != null`.

```
⚠️  MISSING · Last seen HCM · 2 days ago              [Mark as Found]
```

| Field | Display |
|-------|---------|
| `missing_status.reported_at` | Time string following the same display rules as post cards (see `screen_1_home_explore.md` → Post Card → Time display rules). E.g. `"2d"`, `"28 May"` |
| `missing_status.last_seen_location` | `city_code` display, e.g. `"HCM"` |

**"Mark as Found" button:**
- Shown to all family members (owner + parents)
- **Disabled** for parents — only owner can tap it
- Tap (owner) → confirmation: *"Mark Bụi as found?"* → `MarkPetFound mutation (BF)`
- Banner removed immediately on success
- Button re-enables automatically when a new missing report is filed (`pet.missing_status != null`)

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

> **Triggers notification:** when the server-side AI analysis (run after a post is published, see screen_7 → `CreatePost`) detects a possible health concern, it fires a `HEALTH_ALERT` notification to the pet's family/owner (see screen_10 → Notifications tab). This is a server-side async event, not a client call.

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

**"Report Missing" button** (below health content, always visible in Health tab):

```
[📍 Report Missing]
```

- Tap → opens Report Missing bottom sheet (see Section 7)
- Shown to all family members (owner + parents)

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

### 5. Delete Pet Button

```
[Delete Bụi]   ← red/destructive color, bottom of all tabs
```

- Owner only; hidden for non-owner parents
- Tap → confirmation dialog:
  *"Delete Bụi? This pet will be removed from your family's pet list. Posts linked to Bụi will remain. This action can be undone."*
  → Confirm → `DeletePet mutation (BD)` (soft delete: `is_deleted = true`)
- Pet immediately disappears from:
  - Pet switcher (header)
  - My Pets pet rows (Screen 8)
  - Family Posts pet list (Screen 3)
  - Any filter or picker referencing pets
- Posts and media tags are **not modified** — historical posts retain the pet badge (pet still exists in DB, just hidden from active UI)
- Navigate back to My Pets tab after delete

---

### 6. Edit Pet Bottom Sheet

Opened from `[...]` → "Edit Pet". Owner only.

| Field | Required | Notes |
|-------|----------|-------|
| Avatar | No | Replace pet avatar — upload via `RequestMediaUpload (BV)` `{ purpose: "PET_AVATAR" }` (screen_4) → use returned `publicUrl`; no AI scan |
| Name | Yes | Pet display name |
| Public | Yes | Toggle `is_public` on/off; default `true`; when off → pet hidden from Family Posts, not searchable, non-member post card badge treated as random |
| Species | Yes | Read-only if set by AI scan; editable if entered manually |
| Breed | No | Read-only if set by AI scan; editable if entered manually; can be left blank |
| Gender | Yes | `male` / `female` / `unknown` |
| Birthday | No | Date picker |
| Weight | No | Number + unit (kg) |

- Submit → `UpdatePet mutation (BC)`
- `species` and `breed` fields: if set by AI scan, shown as read-only with label *"Set by AI"*; owner can override by unlocking (tap lock icon → confirmation)

---

### 7. Report Missing Bottom Sheet

```
[drag handle]
Report Missing: Bụi                              [× close]
──────────────────────────────────────────────────────────
📍  Last seen location *
    [City / Country picker]

📷  Attach photos/videos (optional)
    [Media picker — max 5, no AI scan]

[Submit Report]
```

| Field | Required | Notes |
|-------|----------|-------|
| `last_seen_location` | Yes | `{ city, city_code, country, country_code }` — same picker as post location |
| `media` | No | Up to 5 items; uploaded files only; no AI scan triggered |

**On submit:**
- `ReportMissing mutation (BE)`
- Pet `missing_status` set: `{ reported_at, last_seen_location, media[] }`
- Pet status in all UI shows `MISSING` badge (amber/red)
- Missing banner appears on Pet Detail (Section 3)
- Push notification sent to all **followers** of this family: *"Bụi from Minh's Family is missing! Last seen in HCM"*

---

## AI Processing Pipeline (background)

Triggered after a post is published that links to a pet (new or existing):

```
Post published → media_tag.type = pet → pet identified/created
  │
  ├─ Health analysis (per post)  [server-side async process — not a client API call]
  │     └─> pet.health_status = "checking" while in queue
  │           └─> Complete → update pet health record + status
  │           └─> No result → pet.health_status = "no_data"
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
query Pet($id: ID!) {
  pet(id: $id) {
    id
    name
    species
    breed
    breedId
    isPublic
    gender
    ageDisplay
    birthday
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
{ "id": "pet_111" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "pet": {
      "id": "pet_111",
      "name": "Bụi",
      "species": "Cat",
      "breed": "Orange Tabby Cat",
      "breedId": "breed_orange_tabby_cat",
      "isPublic": true,
      "gender": "MALE",
      "ageDisplay": "3 years",
      "birthday": "2023-01-15",
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

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_FAMILY_MEMBER` | Caller is not a member of this pet's family |
| `404` | `PET_NOT_FOUND` | Pet does not exist or is soft-deleted |

---

### BB. Query: `FamilyPets`

Fetch all active (non-deleted) pets for the family — used to populate the pet switcher.
**Auth:** Required (family member)

**Operation:**
```graphql
query FamilyPets($familyId: ID!) {
  familyPets(familyId: $familyId) {
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
    "familyPets": [
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
mutation UpdatePet($id: ID!, $input: UpdatePetInput!) {
  updatePet(id: $id, input: $input) {
    id
    name
    species
    breed
    breedId
    isPublic
    gender
    ageDisplay
    birthday
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
  "id": "pet_111",
  "input": {
    "name": "Bụi Bụi",
    "species": "Cat",
    "isPublic": true,
    "gender": "MALE",
    "birthday": "2023-01-15",
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
      "species": "Cat",
      "breed": "Orange Tabby Cat",
      "breedId": "breed_orange_tabby_cat",
      "isPublic": true,
      "gender": "MALE",
      "ageDisplay": "3 years",
      "birthday": "2023-01-15",
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
mutation DeletePet($id: ID!) {
  deletePet(id: $id) {
    success
  }
}
```

**Variables:**
```json
{ "id": "pet_111" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "deletePet": {
      "success": true
    }
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

> **Triggers notifications** (see screen_10 → Notifications tab): a `PET_MISSING` notification to the family's members/followers, and a `MISSING_NEARBY` notification to other users near the last-seen location.

**Operation:**
```graphql
mutation ReportMissing($petId: ID!, $input: MissingReportInput!) {
  reportMissing(petId: $petId, input: $input) {
    id
    petId
    reportedAt
    lastSeenLocation {
      city
      cityCode
      country
      countryCode
    }
    mediaUrls
  }
}
```

**Variables:**
```json
{
  "petId": "pet_111",
  "input": {
    "lastSeenLocation": {
      "city": "Hồ Chí Minh",
      "cityCode": "HCM",
      "country": "Việt Nam",
      "countryCode": "VN"
    },
    "mediaUrls": [
      "https://cdn.petapp.com/media/tmp_missing_001.jpg"
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
      "petId": "pet_111",
      "reportedAt": "2026-06-06T10:00:00Z",
      "lastSeenLocation": {
        "city": "Hồ Chí Minh",
        "cityCode": "HCM",
        "country": "Việt Nam",
        "countryCode": "VN"
      },
      "mediaUrls": ["https://cdn.petapp.com/media/tmp_missing_001.jpg"]
    }
  }
}
```

**Side effects:**
- `pet.missingStatus` set
- Push notification dispatched to family followers

---

### BF. Mutation: `MarkPetFound`

Mark a missing pet as found.
**Auth:** Required (owner only)

> **Triggers notification:** fires a `PET_FOUND` notification (see screen_10 → Notifications tab) to the family's members/followers and to users who were notified of the original `MISSING_NEARBY`.

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
- Optional push notification: *"Good news! Bụi from Minh's Family has been found 🎉"*

---

## User Flow Diagrams

### Open Pet Detail

```
User taps pet row in My Pets
  └─> Pet query (BA) + FamilyPets query (BB)
        └─> Render pet identity + missing banner (if applicable)
              └─> Default tab: Health
                    ├─ status=checking → show spinner
                    ├─ status=no_data  → show "No data" message
                    └─ status=result   → render HTML blob
```

### Switch Pet

```
User taps another pet avatar in switcher
  └─> Pet query (BA) { id: other_pet_id }
        └─> Re-render all sections with new pet data
            (stay on same tab)
```

### Report Missing

```
User taps [Report Missing]
  └─> Open bottom sheet
        └─> Fill location (required) + attach media (optional)
              └─> ReportMissing mutation (BE)
                    └─> success → close sheet → missing banner appears
                               push notification sent to followers
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
| Health tab `source_post_id` | Tapping the source post thumbnail → navigate to Post Detail |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Delete type | Soft delete (`is_deleted = true`); pet hidden from active UI; posts and tags unchanged |
| 2 | Media tags on deleted pet | Not modified — historical posts retain pet badge (pet still exists in DB) |
| 3 | Tab content format | All tabs render pre-generated HTML blob; client does not parse |
| 4 | Health data scope | Per-post; only latest post's analysis is shown in Health tab |
| 5 | Food/Behavior/Med/Vac scope | Per-breed; shared across all pets of same breed |
| 6 | Tab status independence | Each tab has its own status; tabs update independently as AI completes each task |
| 7 | Report Missing access | All family members (owner + parents) can report missing |
| 8 | Mark as Found access | Owner only |
| 9 | Missing push notification | Sent to family followers on report; optional "found" notification on resolution |
| 10 | `[+]` in pet switcher | Creates post (pet created via AI scan); same as normal Create Post flow |
