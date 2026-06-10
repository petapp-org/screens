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
| Uploaded **photo** | ✅ AI Scan button | AI scan sets the initial tag; user can override (§4) |
| Uploaded **video** | ❌ none | **Manual tag only** — no AI scan on video (§4, manual mode) |
| **Embedded** URL (YouTube/Vimeo) | ❌ none | Auto-tagged `Random` |

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

> **Photos only.** Uploaded **videos are not AI-scanned** — they are tagged manually (§4, manual mode). Embedded media is auto-`random`. `ScanMedia (AT)` is never called for videos or embeds.

**Purpose:** detect if a pet is present in the media, identify its species and breed, then attempt to match with named pets in the current family.

```
User taps [AI Scan] on a media item
  └─> ScanMedia mutation (AT)  { mediaUrl, familyId }
        └─> (loading state on thumbnail)
              ├─ Pet detected + match found in family
              │     └─> tag: { type=pet, id=pet_xxx, species="cat", breed="British Shorthair" }
              │           Badge shows: [pet avatar]  pet name  ✓
              ├─ Pet detected + no match + breed known
              │     └─> tag: { type=random, id=null, species="cat", breed="British Shorthair" }
              │           Badge shows: "British Shorthair"  (with edit pencil icon)
              ├─ Pet detected + no match + breed unknown
              │     └─> tag: { type=random, id=null, species="cat", breed=null }
              │           Badge shows: "cat"  (with edit pencil icon)
              └─ No animal detected
                    └─> tag: { type=random, id=null, species=null, breed=null }
                          Badge shows: "Random"  (with edit pencil icon)
```

**After scan, user can manually override the tag** via the tag edit flow (see below).

---

### 4. Tag Edit Flow (manual override)

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
| Color | No | Text input (can pre-fill from AI scan `color` if returned) |
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

### AT. Mutation: `ScanMedia`
Scan an **already-uploaded** media item for pet detection. The media is first uploaded via `RequestMediaUpload (BV)` (screen_4) — `mediaUrl` here is the `publicUrl` it returned. **Pet detection runs on uploaded photos only** — **videos and embeds are never scanned** (videos are tagged manually, embeds are auto-`random`); avatars (user/family/pet) are uploaded the same way but never scanned.
**Auth:** Required

**Operation:**
```graphql
mutation ScanMedia($input: ScanMediaInput!) {
  scanMedia(input: $input) {
    detected
    species
    breed
    color
    matchedPet {
      id
      name
      avatarUrl
    }
  }
}
```

**Variables:**
```json
{
  "input": {
    "mediaUrl": "https://cdn.petapp.com/media/tmp_upload_001.jpg",
    "familyId": "fam_xyz",
    "skipPetMatch": false
  }
}
```

**Response `200 OK`:**

Pet detected + matched to family pet:
```json
{
  "data": {
    "scanMedia": {
      "detected": true,
      "species": "Cat",
      "breed": "Orange Tabby Cat",
      "color": "orange",
      "matchedPet": {
        "id": "pet_111",
        "name": "Pudding",
        "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg"
      }
    }
  }
}
```

Pet detected + no family match:
```json
{
  "data": {
    "scanMedia": {
      "detected": true,
      "species": "Cat",
      "breed": "British Shorthair",
      "color": "grey",
      "matchedPet": null
    }
  }
}
```

No pet detected:
```json
{
  "data": {
    "scanMedia": {
      "detected": false,
      "species": null,
      "breed": null,
      "color": null,
      "matchedPet": null
    }
  }
}
```

**Notes:**
- `color` is returned by the AI for use in the Create Pet form (pre-fill). It is **not stored** in `mediaTag`.
- Resulting `mediaTag` written to the post uses `{ type, id, species, breed }` (Screen 1 canonical structure).
- In Random Pets context (`[+]` from Screen 8), call with `familyId` omitted or `skipPetMatch: true` — `matchedPet` is always `null`.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_MEDIA_URL` | URL is not reachable or not a supported media format |
| `404` | `FAMILY_NOT_FOUND` | `familyId` does not exist |
| `504` | `AI_TIMEOUT` | AI scan service did not respond in time — client should retry |

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
    species
    breed
    isPublic
    gender
    ageMonths
    avatarUrl
  }
}
```

> Tuổi trả về dạng `ageMonths` (Int) — client tự format hiển thị theo locale + `birthDatePrecision`, server không format chuỗi.

**Variables:**
```json
{
  "familyId": "fam_xyz",
  "input": {
    "name": "Snowball",
    "species": "Cat",
    "breed": "British Shorthair",
    "isPublic": true,
    "gender": "FEMALE",
    "birthday": "2024-01-15",
    "weightKg": 3.2,
    "avatarUrl": "https://cdn.petapp.com/media/tmp_pet_avatar.jpg"
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
      "species": "Cat",
      "breed": "British Shorthair",
      "isPublic": true,
      "gender": "FEMALE",
      "ageMonths": 12,
      "avatarUrl": "https://cdn.petapp.com/media/tmp_pet_avatar.jpg"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_FAMILY_MEMBER` | Caller is not a member of the given family |
| `404` | `FAMILY_NOT_FOUND` | Family does not exist |
| `422` | `INVALID_SPECIES` | Species value not found in DB |
| `422` | `INVALID_BREED` | Breed does not belong to the given species |
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
```json
{
  "input": {
    "familyId": "fam_xyz",
    "caption": "Pudding nằm chờ mama nấu cơm 🌕",
    "privacy": "PUBLIC",
    "location": {
      "city": "Hồ Chí Minh",
      "cityCode": "HCM",
      "country": "Việt Nam",
      "countryCode": "VN"
    },
    "media": [
      {
        "url": "https://cdn.petapp.com/media/tmp_upload_001.jpg",
        "type": "UPLOADED",
        "order": 1,
        "mediaTag": {
          "type": "PET",
          "id": "pet_111",
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

> **Triggers notification:** fires a `FAMILY_NEW_POST` notification to the family's **followers** (see screen_22 — Notifications screen), respecting post privacy (a `private` post notifies only family members; `followers`/`public` notify followers).
>
> **AI side-effects:** the post's media is scanned server-side; this may later raise a `HEALTH_ALERT` notification to the pet owner if a health concern is detected (see screen_9).

**Operation:**
```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    caption
    privacy
    location {
      city
      cityCode
      country
      countryCode
    }
    media {
      url
      type
      order
      mediaTag {
        type
        id
        species
        breed
      }
    }
    createdAt
  }
}
```

**Variables:**
```json
{
  "input": {
    "familyId": "fam_xyz",
    "caption": "Pudding nằm chờ mama nấu cơm 🌕",
    "privacy": "PUBLIC",
    "location": {
      "city": "Hồ Chí Minh",
      "cityCode": "HCM",
      "country": "Việt Nam",
      "countryCode": "VN"
    },
    "media": [
      {
        "url": "https://cdn.petapp.com/media/tmp_upload_001.jpg",
        "type": "UPLOADED",
        "order": 1,
        "mediaTag": {
          "type": "PET",
          "id": "pet_111",
          "species": "Cat",
          "breed": "Orange Tabby Cat"
        }
      },
      {
        "url": "https://www.youtube.com/watch?v=abc123",
        "type": "EMBEDDED",
        "order": 2,
        "mediaTag": {
          "type": "RANDOM",
          "id": null,
          "species": null,
          "breed": null
        }
      }
    ]
  }
}
```

**Response `201 Created`:** Full `Post` object (canonical shape from Screen 1).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `NO_MEDIA` | Post has no media items |
| `400` | `MEDIA_UNTAGGED` | One or more media items have no tag |
| `400` | `TOO_MANY_MEDIA` | More than 10 media items |
| `400` | `MULTIPLE_EMBEDDED` | More than 1 embedded URL |
| `403` | `NOT_FAMILY_MEMBER` | User is not a member of the specified family |

---

### BQ. Query: `ListSpecies`

Fetch all available species options for the Create Pet form.  
**Auth:** Not required

```graphql
query ListSpecies {
  listSpecies {
    id
    name
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "listSpecies": [
      { "id": "species_cat", "name": "Cat" },
      { "id": "species_dog", "name": "Dog" },
      { "id": "species_bird", "name": "Bird" }
    ]
  }
}
```

---

### BR. Query: `ListBreeds`

Fetch all breed options for a given species, for the Create Pet form.  
**Auth:** Not required

```graphql
query ListBreeds($speciesId: ID!) {
  listBreeds(speciesId: $speciesId) {
    id
    name
  }
}
```

**Variables:** `{ "speciesId": "species_cat" }`

**Response `200 OK`:**
```json
{
  "data": {
    "listBreeds": [
      { "id": "breed_british_shorthair", "name": "British Shorthair" },
      { "id": "breed_orange_tabby", "name": "Orange Tabby Cat" }
    ]
  }
}
```

**Notes:**
- Called after user selects a species to populate the breed dropdown
- Breed list is filtered to the selected species
- If user changes species, breed selection is reset

---

## User Flow Diagrams

### Standard Create Post

```
User opens Create Post (e.g. from fab button)
  └─> Check active family
        ├─ No active family → prompt to set in Profile Settings
        └─ Has active family → load draft (if exists) or empty form
              └─> Add media (upload photo/video or embed)
                    ├─> Uploaded photo → tap AI Scan → ScanMedia mutation (AT)
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
