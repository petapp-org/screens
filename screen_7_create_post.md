# Screen 7: Create Post

## Overview

Form screen for creating a new post under the user's active family.  
Requires login and an active family. Draft is auto-saved throughout; on successful publish the draft is deleted.  
After publish → navigate to **My Pets** tab (active family). Post can only be created under the active family — no family switching mid-flow.

**Prerequisites:**
- User must be logged in
- User must have an active family (is owner or parent of that family)
- If no active family → show prompt to set one in Profile Settings

---

## UI Layout

```
[Header]
  Left: Cancel (× — discards draft after confirmation)
  Center: "New Post"
  Right: "Post" button (disabled until all media are tagged)

[Scrollable area]
  ├── Active family row
  │     └── Family avatar + family name + [switch family — if user is in multiple]
  │
  ├── Media section
  │     ├── [Add Media button] — opens picker (upload or embed URL)
  │     ├── Media grid (thumbnails, reorderable by drag)
  │     │     Each thumbnail:
  │     │       - Remove (×) button
  │     │       - Tag badge (bottom): pet name | breed | "Random"
  │     │       - [AI Scan button] — per uploaded media
  │     └── Counter: "N / 10 media"
  │
  ├── Caption field
  │     └── Multiline text input — "Write a caption..."
  │
  ├── Location field
  │     └── [📍 icon]  "Add location"  → city/country picker
  │         When set: shows "HCM - VN" with clear (×) button
  │
  ├── Privacy field
  │     └── [🔒 icon]  dropdown: Public | Followers only | Family only
  │         Pre-filled from family's defaultPrivacy
  │
  └── (spacer)

[Fixed bottom: "Post" button — full width, disabled until all media tagged]
```

---

## Components

### 1. Active Family Row

- Shows current active family (avatar + name)
- If user is a member of multiple families: shows a **"Switch"** link
  - Tap → bottom sheet listing all families the user belongs to
  - Select → updates the active family for this post only (does not change global active family setting)
- If user has no active family → show "Set active family" → navigate to Profile Settings

---

### 2. Media Section

**Max 10 media items per post. Min 1 required.**

**Media kinds & tagging:**

| Kind | AI Scan | How it's tagged |
|------|---------|-----------------|
| Uploaded **photo** | ✅ AI Scan button — **opt-in, quota-gated** (§3a) | Default **untagged → user tags manually** (§4). User may spend 1 AI scan to auto-fill the tag; can override after (§4) |
| Uploaded **video** | ❌ none | **Manual tag only** — no AI scan on video (§4, manual mode) |
| **Embedded** URL (YouTube/Vimeo) | ❌ none | Auto-tagged `Random` |

> **AI scan is opt-in (changed — was always-available).** To protect AI capacity at scale, pet detection is no longer run automatically/freely on every photo. Each user has a limited **AI scan quota** (default **3** for a new account, admin-configurable — §3a). Photos are tagged **manually by default** (§4); the user explicitly spends quota per photo to get AI assistance. Videos/embeds never consume quota.

**Uploaded video constraints** (validated on selection; over-limit → reject with a message):
- Max **duration: 90 seconds**
- Max **file size: 100 MB**
- Formats: **MP4 / MOV** (H.264)

**Add Media button:** opens action sheet:
- "Upload from device" → image/video file picker (multiple selection allowed, up to remaining slots)
- "Embed URL" → text input for YouTube/Vimeo URL (only 1 embedded URL allowed per post; button hidden once one is added)

**Uploaded photo thumbnail:**
- Image preview · `×` remove (top-right) · Tag badge (bottom-left) · **AI Scan button** (bottom-right)

**Uploaded video thumbnail:**
- Video thumbnail with a **duration overlay** (e.g. `0:42`) + play icon · `×` remove · Tag badge (bottom-left)
- **No AI Scan button** — tapping the tag badge opens the manual tag sheet directly (§4)

**Embedded media:**
- Shows video thumbnail (fetched from YouTube/Vimeo API)
- `×` remove button
- Tag badge: always shows **"Random"** (embedded URLs are auto-tagged as `random`, no AI scan)
- No AI Scan button

**Reorder:** drag-and-drop to rearrange media items. Order determines carousel order on the post.

---

### 3. AI Scan Flow (per uploaded **photo**)

> **Photos only.** Uploaded **videos are not AI-scanned** — they are tagged manually (§4, manual mode). Embedded media is auto-`random`. `IdentifyPetFromMedia (AT)` is never called for videos or embeds.

#### 3a. AI Scan Quota (opt-in gate) — ⏳ GAP, pending petapp-be issue

> **Status:** the quota model below is **not yet in the contract**. `me { aiScan { … } }` and quota enforcement on the scan mutations do **not** exist in `schema.graphql` today — this section specifies the target. Tracked by **petapp-be#TBD** (open before client build). Until shipped, the client may treat quota as unlimited or hide the scan affordance.

- **Quota unit:** **1 scan = 1 photo.** Each tap on `[AI Scan]` for one photo consumes **1 quota** and runs **both** AI steps for that photo as a single billable action: **(1) pet detection** (`identifyPetFromMedia (AT)`) **+ (2) health analysis** (`requestHealthCheck (AZ)`, only when the photo resolves to a pet — needs a `petId`). The two steps together count as **one** quota unit.
- **Default grant:** new accounts get **3** scans (admin-configurable default; admin can top-up per user — backend/admin concern).
- **Display:** show remaining quota in two places, both loaded from `me.aiScan.remaining` (query **AY**):
  - Media section header: `✨ {remaining} AI scans left`
  - Each photo's scan button: `[✨ Scan · {remaining} left]`
- **Decrement:** only on **successful** detection. On `AI_TIMEOUT (504)` → **refund / no decrement**, allow retry.
- **Exhausted (`remaining = 0`):** scan button **disabled** on all photos; header shows *"No AI scans left"* (+ upgrade entry point, future). User falls back to manual tagging (§4).
- **Random media (no pet match):** detection still runs (consumes the quota), but the health step is **skipped** — `requestHealthCheck` requires a `petId`, which a random/unmatched frame does not have.

> **Other AI endpoints** (`generatePostCaption`, `requestMoodAnalysis`) are also AI-cost but are **out of scope** for this quota in v1. Flag for backend: they should be rate-limited or folded into a broader AI-quota scheme separately (petapp-be#TBD).

#### 3b. Scan behaviour

**Purpose:** detect if a pet is present in the media, identify its species and breed, then attempt to match with named pets in the current family. **Matching is client-side**: the mutation returns raw AI output (`speciesId`, `breedId`, confidence scores, `rawLabel`, `color`); the client so sánh kết quả với danh sách pets của user để tự xác định khớp.

```
User taps [✨ Scan · N left] on a photo   (only when remaining > 0)
  └─> consumes 1 quota → IdentifyPetFromMedia (AT)  { mediaId }
        │  (then, if result resolves to a pet: requestHealthCheck (AZ) { mediaId, petId }
        │   fires async — same quota unit; health populates Pet Detail later, see screen_9)
        ▼
  (the detection branches below set the initial tag; user can still override via §4)

User taps [AI Scan] on a media item
  └─> IdentifyPetFromMedia mutation (AT)  { mediaId }
        └─> (loading state on thumbnail)
              ├─ speciesConfidence high + breedConfidence high
              │     → client matches speciesId/breedId against user's pets
              │     ├─ match found in family
              │     │     └─> tag: { type=pet, petId=pet_xxx, species="cat", breed="British Shorthair" }
              │     │           Badge shows: [pet avatar]  pet name  ✓
              │     └─ no family pet match
              │           └─> tag: { type=random, petId=null, species="cat", breed="British Shorthair" }
              │                 Badge shows: "British Shorthair"  (with edit pencil icon)
              ├─ speciesConfidence high + breedConfidence low (breedId empty)
              │     └─> tag: { type=random, petId=null, species="cat", breed=null }
              │           Badge shows: "cat"  (with edit pencil icon)
              └─ speciesConfidence low (no animal detected)
                    └─> tag: { type=random, petId=null, species=null, breed=null }
                          Badge shows: "Random"  (with edit pencil icon)
```

**After scan, user can manually override the tag** via the tag edit flow (see below).

---

### 4. Tagging media (manual is the default path)

With AI scan now opt-in (§3a), **most media are tagged manually**. There are two manual entry points: a fast **post-level pet picker** (§4a, for the common "this post is about my pet" case) and the **per-frame tag sheet** (§4b, for overrides and mixed posts).

#### 4a. Quick post-level pet picker (map the whole post to a pet)

Shown above the media grid whenever the post has ≥ 1 untagged frame:

```
🐾 Who's in this post?
   ( ◯ Pudding )  ( ◯ Mochi )  ( + New pet )    ← chips from active family's pets
```

- **Single-select a pet** → applies `{ type=pet, petId, species, breed }` (species/breed pulled from the pet record) to **every untagged frame** in the post — one tap tags the whole post. (Frames already tagged individually via §4b are left untouched.)
- **Pre-selection:** the family's **primary pet** (or the most-recently-posted pet) is pre-highlighted to cut friction — user just confirms.
- **`+ New pet`** → opens the Create Pet sub-form (§4b, manual mode — species/breed picked from DB option lists, no AI pre-fill).
- This needs **no AI and no quota** — pets already in the family carry their own species/breed.
- For a **multi-pet post** (different pet per frame), use §4b per-frame instead.

> **Random animals** (e.g. a street cat that isn't a family pet): leave the frame as `random`. Without AI, a random tag has `species = breed = null`, so it will **not** surface in the Random Pets feed (`screen_14`, which needs a detected breed/species). If the user wants it to appear there, they can manually pick a species/breed in the §4b sheet.

#### 4b. Per-frame tag sheet (override / mixed posts)

Triggered by tapping the tag badge or edit (pencil) icon on any media item.

> **Uploaded videos** (no AI scan) open this sheet directly in **manual mode** — identical to the *"no animal detected"* case below: select an existing pet, create a new pet (species/breed chosen manually from the DB option lists), or mark as Random. There is no AI pre-fill for videos.

Opens a bottom sheet:

```
[Bottom sheet]
  "Tag this media"

  ── Select existing pet ──
  [Pet avatar]  Pudding · Orange Tabby Cat  ← tap to select
  [Pet avatar]  Mochi · Vietnamese Native   ← tap to select

  ── Or create new pet ──
  [+ Create new pet]  →  opens Create Pet sub-form

  ── Or ──
  [Mark as Random]
```

Actions available depend on the current tag state:

**When `type=pet` (matched pet):**
- Keep as-is (close sheet)
- Select a different existing pet → tag updated to new `id`
- Unlink → tag set to `{ type=random, id=null, breed=<AI-detected breed if any> }`

**When `type=random, breed != null` (AI detected breed + species, no match):**
- Keep as random (close sheet)
- Select existing pet → tag set to `{ type=pet, id=pet_xxx, species=..., breed=pet's_breed }`
- Create new pet (species + breed pre-filled from scan):

| Field | Required | Notes |
|-------|----------|-------|
| Name | Yes | Pet's display name |
| Species | Yes | Pre-filled from scan; **select from DB option list** (not free text) |
| Breed | No | Pre-filled from scan; **select from DB option list** filtered by selected species |
| Color | No | Text input — **pre-filled from scan** when available (`IdentifyPetResult.color`; see AT Notes). Empty when scanned media had no colour or for manual/video tagging |
| Gender | Yes | `male` / `female` / `unknown` |
| Birthday | No | Date picker |
| Weight | No | Number + unit (kg) |
| Avatar | No | Upload photo or use a frame from this media |

  Submit → `CreatePet mutation (AU)` → tag set to `{ type=pet, id=newPetId, species=..., breed=... }`

**When `type=random, breed=null, species != null` (AI detected species only, no breed, no match):**
- Keep as random (close sheet)
- Select existing pet → tag set to `{ type=pet, id=pet_xxx, species=..., breed=pet's_breed }`
- Create new pet (species pre-filled, breed left blank):
  - Same form as above; `species` pre-filled, `breed` empty and optional
  - Submit → `CreatePet mutation (AU)`

**When `type=random, species=null, breed=null` (no animal detected):**
- Keep as random (close sheet)
- Select existing pet manually → tag set to `{ type=pet, id=pet_xxx, species=..., breed=pet's_breed }`
- Create new pet manually — form shows message: *"We couldn't detect an animal in this media. Please fill in the details manually."*
  - `species` and `breed` fields are **not pre-filled**; user selects from DB-backed option lists (not free text input)
  - `species` is required; `breed` is optional (leave blank if unknown)
  - All other fields same as above

---

### 5. Caption Field

- Optional, multiline, no character limit
- Keyboard auto-opens after media is added (optional UX behaviour)

---

### 6. Location Field

- Optional
- Tap → opens city/country picker (searchable dropdown or device location)
- Returns: `{ city, cityCode, country, countryCode }`
- Displayed on field as `"HCM - VN"` once set
- Tap × to clear

---

### 7. Privacy Field

- Dropdown: **Public** | **Followers only** | **Family only**
- Default: loaded from active family's `defaultPrivacy`
- Applies to this post only; does not change the family's default

---

### 8. Draft Auto-save

- Draft is saved automatically as the user fills in the form (debounced, every 3s of inactivity)
- `SaveDraft mutation (AV)` with current form state
- If user taps Cancel:
  - Confirmation dialog: "Discard post?" → Discard / Keep editing
  - Discard → `DeleteDraft mutation (AW)` → navigate back
  - Keep editing → dismiss dialog
- Draft is deleted automatically after successful publish

---

### 9. Post Button

- Located in header (right) and as fixed full-width button at bottom
- **Disabled** until:
  - At least 1 media added
  - All media items have a tag (any `type`: `pet` or `random`, with or without `breed`)
- On tap → validate → `CreatePost mutation (AX)` → on success, delete draft → navigate to My Pets tab (active family)

---

## API Endpoints Required

### AT. Mutation: `IdentifyPetFromMedia`
Scan an **already-uploaded** media item for pet detection. The media is first uploaded via `SignUploadBatch (BV)` (screen_4) — `mediaId` here is the id of the uploaded media record. **Pet detection runs on uploaded photos only** — **videos and embeds are never scanned** (videos are tagged manually, embeds are auto-`random`); avatars (user/family/pet) are uploaded the same way but never scanned.
**Auth:** Required

**Operation:**
```graphql
mutation IdentifyPetFromMedia($mediaId: ID!) {
  identifyPetFromMedia(mediaId: $mediaId) {
    speciesId
    speciesConfidence
    breedId
    breedConfidence
    rawLabel
    color
    colorConfidence
  }
}
```

**Variables:**
```json
{
  "mediaId": "media_tmp_001"
}
```

**Response `200 OK`:**

Pet detected — high confidence (client so sánh `speciesId`/`breedId` với pets của user để xác định khớp; contract chỉ trả raw AI output):
```json
{
  "data": {
    "identifyPetFromMedia": {
      "speciesId": "species_cat",
      "speciesConfidence": 0.97,
      "breedId": "breed_orange_tabby",
      "breedConfidence": 0.88,
      "rawLabel": "orange tabby cat",
      "color": "orange",
      "colorConfidence": 0.82
    }
  }
}
```

Pet detected — breed not recognised (low breedConfidence, breedId empty):
```json
{
  "data": {
    "identifyPetFromMedia": {
      "speciesId": "species_cat",
      "speciesConfidence": 0.91,
      "breedId": "",
      "breedConfidence": 0.21,
      "rawLabel": "grey cat",
      "color": "grey",
      "colorConfidence": 0.6
    }
  }
}
```

No animal detected (all confidences near zero):
```json
{
  "data": {
    "identifyPetFromMedia": {
      "speciesId": "",
      "speciesConfidence": 0.04,
      "breedId": "",
      "breedConfidence": 0.0,
      "rawLabel": "indoor scene",
      "color": null,
      "colorConfidence": null
    }
  }
}
```

**Field docs:**

| Field | Type | Description |
|-------|------|-------------|
| `speciesId` | `String!` | ID of the detected species (e.g. `"species_cat"`); empty string when no animal detected |
| `speciesConfidence` | `Float!` | AI confidence score for the species classification (0–1) |
| `breedId` | `String!` | ID of the detected breed (e.g. `"breed_orange_tabby"`); empty string when breed is unknown |
| `breedConfidence` | `Float!` | AI confidence score for the breed classification (0–1) |
| `rawLabel` | `String!` | Raw label from the AI model (for debugging / display fallback) |
| `color` | `String` | Detected coat colour (e.g. `"orange"`); `null` when no animal detected / colour not determined |
| `colorConfidence` | `Float` | AI confidence for the colour (0–1); `null` when `color` is `null` |

**Notes:**
- **Client-side matching:** so sánh `speciesId`/`breedId` trả về với danh sách pets của user để tìm khớp (e.g. một pet có cùng speciesId và breedId). Contract chỉ trả raw AI output — không có `matchedPet` hay logic matching phía server.
- **Color pre-fill (available):** contract `IdentifyPetResult` **does** carry `color` + `colorConfidence` (verified against `schema.graphql`), so the Create Pet form can pre-fill the coat colour from the scan. *(Earlier spec drift claimed `color` was removed — that was wrong; corrected here.)*
- **Quota (⏳ GAP):** this mutation is the billable detection step gated by the AI scan quota (§3a). Quota field/enforcement are **not yet in the contract** — pending petapp-be#TBD. When `remaining = 0`, the client must not call this mutation (button disabled); the target backend behaviour is to reject with `QUOTA_EXCEEDED` as a safety net.
- Resulting `mediaTag` written to the post uses `{ type, petId, species, breed }` (canonical MediaTag structure (#940/ADR-0027)).
- In Random Pets context (`[+]` from Screen 8), call without a `familyId` context — matching against family pets is skipped; client treats all results as unmatched.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_MEDIA_ID` | Media ID does not exist or is not a supported media format |
| `403` | `QUOTA_EXCEEDED` | ⏳ GAP (pending petapp-be#TBD) — user has no AI scan quota left (§3a) |
| `404` | `FAMILY_NOT_FOUND` | Associated family does not exist |
| `504` | `AI_TIMEOUT` | AI scan service did not respond in time — client should retry (no quota decrement) |

---

### AU. Mutation: `CreatePet`
Create a new pet under a family.  
**Auth:** Required (owner or parent of the family)

**Operation:**
```graphql
mutation CreatePet($familyId: ID!, $input: CreatePetInput!) {
  createPet(familyId: $familyId, input: $input) {
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
    sex
    ageMonths
    avatarUrl
  }
}
```

> Tuổi trả về dạng `ageMonths` (Int) — client tự format hiển thị theo locale + `birthDatePrecision`, server không format chuỗi.

**Variables:**

`CreatePetInput` fields (ADR-0024): `name`, `speciesId` (ID!), `sex` (PetSex!), `breedId` (String, optional), `microchipNumber` (String, optional), `birthDate` (String, optional), `birthDatePrecision` (BirthDatePrecision, default UNSPECIFIED), `weightKg` (String, optional), `primaryMediaId` (String, optional — ID từ `signUploadBatch`).

```json
{
  "familyId": "fam_xyz",
  "input": {
    "name": "Snowball",
    "speciesId": "species_cat",
    "breedId": "breed_british_shorthair",
    "sex": "FEMALE",
    "birthDate": "2024-01-15",
    "weightKg": "3.2",
    "primaryMediaId": "media_tmp_pet_avatar_001"
  }
}
```

**Response `201 Created`:**
```json
{
  "data": {
    "createPet": {
      "id": "pet_222",
      "name": "Snowball",
      "species": { "name": "Cat", "iconEmoji": "🐱" },
      "breed": { "nameVi": "Mèo lông ngắn Anh", "nameEn": "British Shorthair" },
      "sex": "FEMALE",
      "ageMonths": 12,
      "avatarUrl": "https://cdn.petapp.com/pets/pet_222/avatar.jpg"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_FAMILY_MEMBER` | Caller is not a member of the given family |
| `404` | `FAMILY_NOT_FOUND` | Family does not exist |
| `422` | `INVALID_SPECIES` | `speciesId` not found in DB |
| `422` | `INVALID_BREED` | `breedId` does not belong to the given species |
| `422` | `NAME_REQUIRED` | Pet name is missing |

---

### AV. Mutation: `SaveDraft`
Save or update the current post draft.  
**Auth:** Required

**Operation:**
```graphql
mutation SaveDraft($input: CreatePostInput!) {
  saveDraft(input: $input) {
    draftId
  }
}
```

**Variables:**

`PostLocationInput` chỉ nhận `cityCode` (String!, bắt buộc), `lat` (Float, optional), `lng` (Float, optional). Display names (`city`, `country`, `countryCode`) được resolve server-side từ `cityCode`.

```json
{
  "input": {
    "familyId": "fam_xyz",
    "body": "Pudding nằm chờ mama nấu cơm 🌕",
    "visibility": "PUBLIC",
    "location": {
      "cityCode": "HCM"
    },
    "media": [
      {
        "order": 1,
        "sourceType": "UPLOADED",
        "mediaId": "media_001",
        "mediaTag": {
          "type": "PET",
          "petId": "pet_111",
          "species": "Cat",
          "breed": "Orange Tabby Cat"
        }
      }
    ]
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "saveDraft": {
      "draftId": "draft_abc"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_FAMILY_MEMBER` | User is not a member of the specified family |
| `404` | `FAMILY_NOT_FOUND` | Family does not exist |

---

### AW. Mutation: `DeleteDraft`
Delete the current draft (on Cancel).  
**Auth:** Required

**Operation:**
```graphql
mutation DeleteDraft($draftId: ID!) {
  deleteDraft(draftId: $draftId)
}
```

**Variables:**
```json
{
  "draftId": "draft_abc"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "deleteDraft": true
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `DRAFT_NOT_FOUND` | Draft ID does not exist or already deleted |

---

### AX. Mutation: `CreatePost`
Publish the post.  
**Auth:** Required

> **Triggers notification:** fires a `FAMILY_NEW_POST` notification to the family's **followers** (see screen_22 — Notifications screen), respecting post privacy (a `family_only` post notifies only family members; `followers`/`public` notify followers).
>
> **No automatic AI on publish (changed).** Publishing does **not** trigger any AI scan. Pet detection **and** health analysis run only when the user explicitly spends an AI scan on a photo in the editor (§3, `IdentifyPetFromMedia (AT)` + `requestHealthCheck (AZ)`). A `HEALTH_SIGNAL` notification (see screen_9 / screen_22) therefore originates from that opt-in scan, not from `CreatePost`.

**Operation:**
```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    body
    visibility
    location {
      city
      cityCode
      country
      countryCode
    }
    media {
      order
      sourceType
      embedUrl
      embedProvider
      mediaTag {
        type
        petId
        species
        breed
      }
      media {
        id
      }
    }
    createdAt
  }
}
```

**Variables:**

`PostLocationInput` chỉ nhận `cityCode` (String!, bắt buộc), `lat` (Float, optional), `lng` (Float, optional). Display names (`city`, `country`, `countryCode`) được resolve server-side từ `cityCode`.

```json
{
  "input": {
    "familyId": "fam_xyz",
    "body": "Pudding nằm chờ mama nấu cơm 🌕",
    "visibility": "PUBLIC",
    "location": {
      "cityCode": "HCM"
    },
    "media": [
      {
        "order": 1,
        "sourceType": "UPLOADED",
        "mediaId": "media_001",
        "mediaTag": {
          "type": "PET",
          "petId": "pet_111",
          "species": "Cat",
          "breed": "Orange Tabby Cat"
        }
      },
      {
        "order": 2,
        "sourceType": "EMBEDDED",
        "embedUrl": "https://www.youtube.com/watch?v=abc123",
        "embedProvider": "YOUTUBE",
        "mediaTag": {
          "type": "RANDOM",
          "petId": null,
          "species": null,
          "breed": null
        }
      }
    ]
  }
}
```

**Response `201 Created`:** Full `Post` object (canonical shape from Screen 1).

> **Read shape** (`Post.media: [PostMedia!]`) được hoàn tất & expose ở backend qua #778 (read-side ADR-0027).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `NO_MEDIA` | Post has no media items |
| `400` | `MEDIA_UNTAGGED` | One or more media items have no tag |
| `400` | `TOO_MANY_MEDIA` | More than 10 media items |
| `400` | `MULTIPLE_EMBEDDED` | More than 1 embedded URL |
| `403` | `NOT_FAMILY_MEMBER` | User is not a member of the specified family |

---

### BQ. Query: `Species`

Fetch all available species options for the Create Pet form.  
**Auth:** Not required

```graphql
query Species {
  species {
    id
    name
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "species": [
      { "id": "species_cat", "name": "Cat" },
      { "id": "species_dog", "name": "Dog" },
      { "id": "species_bird", "name": "Bird" }
    ]
  }
}
```

---

### BR. Query: `Breeds`

Fetch all breed options for a given species, for the Create Pet form.  
**Auth:** Not required

```graphql
query Breeds($speciesId: ID!) {
  breeds(speciesId: $speciesId) {
    id
    nameVi
    nameEn
  }
}
```

**Variables:** `{ "speciesId": "species_cat" }`

**Response `200 OK`:**
```json
{
  "data": {
    "breeds": [
      { "id": "breed_british_shorthair", "nameVi": "Mèo Anh lông ngắn", "nameEn": "British Shorthair" },
      { "id": "breed_orange_tabby", "nameVi": "Mèo lông ngắn cam", "nameEn": "Orange Tabby Cat" }
    ]
  }
}
```

**Notes:**
- Called after user selects a species to populate the breed dropdown
- Breed list is filtered to the selected species
- If user changes species, breed selection is reset
- `nameVi` / `nameEn` — hiển thị theo locale của user; `BreedGQL` không có field `name` (đã tách thành `nameVi`/`nameEn`)

---

### AY. Query: `MyAiScanQuota` — ⏳ GAP (pending petapp-be#TBD)

Load the user's remaining AI scan quota to drive the `✨ N scans left` UI (§3a) and to enable/disable the scan buttons.

> **Status:** `me.aiScan` does **not** exist in the contract yet. Shape below is the target; open petapp-be#TBD to add it to `type User`. Until then the client hides the counter / treats quota as unlimited.

**Operation (target):**
```graphql
query MyAiScanQuota {
  me {
    id
    aiScan {
      remaining   # Int!  — scans left
      total       # Int!  — total granted (default 3 for new accounts)
      resetAt     # String | null — null = one-time grant, no reset
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `remaining` | `Int!` | Scans left; `0` → scan buttons disabled |
| `total` | `Int!` | Total granted; admin-configurable default (3) + top-up |
| `resetAt` | `String` | ISO timestamp of next reset, or `null` for a non-resetting grant |

---

### AZ. Mutation: `requestHealthCheck` (exists in contract)

Second step of an AI scan (§3a): run health analysis on a scanned photo that resolves to a pet. Already in the contract — `requestHealthCheck(mediaId: ID!, petId: ID!): RequestHealthCheckPayload!`. Returns an async job; results populate Pet Detail's Health tab and may raise `HEALTH_SIGNAL` (see screen_9 / screen_22).

**Operation:**
```graphql
mutation RequestHealthCheck($mediaId: ID!, $petId: ID!) {
  requestHealthCheck(mediaId: $mediaId, petId: $petId) {
    jobId
    status
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `jobId` | `ID!` | Async job handle for the health analysis |
| `status` | `String!` | Initial job status (e.g. `QUEUED`) |

**Notes:**
- Requires a `petId` → only callable after detection (or manual tagging) resolves the frame to a pet. Random/unmatched frames have no `petId` and skip this step.
- Bundled into the **same** quota unit as `IdentifyPetFromMedia (AT)` — see §3a. The quota decrement happens once, on the detection step.
- ⏳ The quota bundling/coordination across the two mutations is **not** modelled in the contract yet (pending petapp-be#TBD). Backend may introduce a combined `scanMedia` mutation so the two steps decrement quota atomically.

---

## User Flow Diagrams

### Standard Create Post

```
User opens Create Post (e.g. from fab button)
  └─> Check active family
        ├─ No active family → prompt to set in Profile Settings
        └─ Has active family → load draft (if exists) or empty form
              └─> Add media (upload photo/video or embed)
                    ├─> Uploaded photo → tap AI Scan → IdentifyPetFromMedia mutation (AT)
                    │         ├─ Match → badge auto-set to pet name
                    │         ├─ Breed detected → badge shows breed (accept or edit)
                    │         └─ No detection → badge "Random" (user can edit)
                    ├─> Uploaded video → no scan → tag manually (§4 manual mode)
                    └─> Embed URL → auto-tagged "Random"
                    └─> [Optional] edit/override any tag manually (§4)
                    └─> Fill caption, location, privacy
                    └─> Tap Post
                          └─> CreatePost mutation (AX)
                                ├─ success → DeleteDraft mutation (AW) → navigate to My Pets tab (active family)
                                └─ error → show validation error
```

### Switch Family Mid-flow

```
User taps "Switch" on family row
  └─> Bottom sheet: list of user's families
        └─> Select different family
              └─> Active family for this post updated
                  (existing media tags may need re-evaluation if pets differ between families)
                  → prompt: "Switching family will clear existing pet tags. Continue?"
                  → Yes → clear all pet tags, keep media files; re-scan recommended
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No active family | Prompt shown before opening form; cannot create post |
| Embedded URL added | Auto-tagged as `random`; no AI Scan button shown; only 1 allowed per post |
| 2nd embedded URL attempt | "Add Embed" button hidden once 1 embedded exists |
| Uploaded **video** added | No AI Scan button; must tag manually (select pet / create pet / Random) before posting |
| Video over 90s or 100 MB | Rejected on selection with a message (e.g. "Video must be 90s or shorter") — not added to the grid |
| Unsupported video format | Rejected on selection (only MP4 / MOV) |
| Media limit reached (10) | "Add Media" button disabled; counter shows "10 / 10" |
| AI scan — no pet detected | Tag: `{ type=random, breed=null }`; user can manually link to existing pet |
| AI scan — breed detected, no family pet match | Tag: `{ type=random, breed="..." }`; user can keep, link to existing pet, or create new pet |
| AI scan — match found | Tag: `{ type=pet, id=pet_xxx, breed="..." }`; user can keep, change pet, or unlink |
| User switches family | Clear all pet-type tags; prompt user to re-tag or re-scan |
| Cancel with filled form | Confirmation dialog; draft preserved on "Keep editing" |
| App crash / close mid-create | Draft auto-saved; restored on next open |
| Post button tapped with untagged media | Show validation: "All media must be tagged before posting" |
