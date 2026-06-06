# Screen 7: Create Post

## Overview

Form screen for creating a new post under the user's active family.  
Requires login and an active family. Draft is auto-saved throughout; on successful publish the draft is deleted.  
After publish → navigate to **My Pets** tab.

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
  │         Pre-filled from family's default_privacy
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

**Add Media button:** opens action sheet:
- "Upload from device" → image/video file picker (multiple selection allowed, up to remaining slots)
- "Embed URL" → text input for YouTube/Vimeo URL (only 1 embedded URL allowed per post; button hidden once one is added)

**Uploaded media thumbnail:**
- Shows image preview or video thumbnail
- `×` remove button (top-right)
- Tag badge (bottom-left): displays current tag status
- **AI Scan button** (bottom-right or overlay): triggers scan for that media

**Embedded media:**
- Shows video thumbnail (fetched from YouTube/Vimeo API)
- `×` remove button
- Tag badge: always shows **"Random"** (embedded URLs are auto-tagged as `random`, no AI scan)
- No AI Scan button

**Reorder:** drag-and-drop to rearrange media items. Order determines carousel order on the post.

---

### 3. AI Scan Flow (per uploaded media)

**Purpose:** detect if a pet is present in the media, identify its breed, then attempt to match with named pets in the current family.

```
User taps [AI Scan] on a media item
  └─> POST /ai/scan-media  { media_url, family_id }
        └─> (loading state on thumbnail)
              ├─ Pet detected + match found in family
              │     └─> tag: { type=pet, id=pet_xxx, breed=pet's_breed }
              │           Badge shows: [pet avatar]  pet name  ✓
              ├─ Pet detected + no match in family
              │     └─> tag: { type=random, id=null, breed="British Shorthair" }
              │           Badge shows: breed name  (with edit pencil icon)
              └─ No pet detected
                    └─> tag: { type=random, id=null, breed=null }
                          Badge shows: "Random"  (with edit pencil icon)
```

**After scan, user can manually override the tag** via the tag edit flow (see below).

---

### 4. Tag Edit Flow (manual override)

Triggered by tapping the tag badge or edit (pencil) icon on any media item.

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

**When `type=random, breed != null` (AI detected breed, no match):**
- Keep as random (close sheet)
- Select existing pet → tag set to `{ type=pet, id=pet_xxx, breed=pet's_breed }`
- Create new pet (breed pre-filled from scan):

| Field | Required | Notes |
|-------|----------|-------|
| Name | Yes | Pet's display name |
| Breed | Yes | Pre-filled from `breed` |
| Color | No | Text input (can pre-fill from AI scan `color` if returned) |
| Gender | Yes | `male` / `female` / `unknown` |
| Birthday | No | Date picker |
| Weight | No | Number + unit (kg) |
| Avatar | No | Upload photo or use a frame from this media |

  Submit → `POST /families/{family_id}/pets` → tag set to `{ type=pet, id=new_pet_id, breed=... }`

**When `type=random, breed=null` (no animal detected):**
- Keep as random (close sheet)
- Select existing pet manually → tag set to `{ type=pet, id=pet_xxx, breed=pet's_breed }`
- *(Cannot create new pet — no breed info available)*

---

### 5. Caption Field

- Optional, multiline, no character limit
- Keyboard auto-opens after media is added (optional UX behaviour)

---

### 6. Location Field

- Optional
- Tap → opens city/country picker (searchable dropdown or device location)
- Returns: `{ city, city_code, country, country_code }`
- Displayed on field as `"HCM - VN"` once set
- Tap × to clear

---

### 7. Privacy Field

- Dropdown: **Public** | **Followers only** | **Family only**
- Default: loaded from active family's `default_privacy`
- Applies to this post only; does not change the family's default

---

### 8. Draft Auto-save

- Draft is saved automatically as the user fills in the form (debounced, every 3s of inactivity)
- `PUT /posts/draft` with current form state
- If user taps Cancel:
  - Confirmation dialog: "Discard post?" → Discard / Keep editing
  - Discard → `DELETE /posts/draft` → navigate back
  - Keep editing → dismiss dialog
- Draft is deleted automatically after successful publish

---

### 9. Post Button

- Located in header (right) and as fixed full-width button at bottom
- **Disabled** until:
  - At least 1 media added
  - All media items have a tag (any `type`: `pet` or `random`, with or without `breed`)
- On tap → validate → `POST /posts` → on success, delete draft → navigate to My Pets tab

---

## API Endpoints Required

### AT. `POST /ai/scan-media`
Scan an uploaded media item for pet detection.  
**Auth:** Required

**Body:**
```json
{
  "media_url": "https://cdn.petapp.com/media/tmp_upload_001.jpg",
  "family_id": "fam_xyz"
}
```

**Response `200 OK`:**

Pet detected + matched to family pet:
```json
{
  "detected": true,
  "breed": "Orange Tabby Cat",
  "color": "orange",
  "matched_pet": {
    "id": "pet_111",
    "name": "Pudding",
    "avatar_url": "https://cdn.petapp.com/pets/pet_111/avatar.jpg"
  }
}
```

Pet detected + no family match:
```json
{
  "detected": true,
  "breed": "British Shorthair",
  "color": "grey",
  "matched_pet": null
}
```

No pet detected:
```json
{
  "detected": false,
  "breed": null,
  "color": null,
  "matched_pet": null
}
```

**Notes:**
- `color` is returned by the AI for use in the Create Pet form (pre-fill). It is **not stored** in `media_tag`.
- Resulting `media_tag` written to the post uses only `{ type, id, breed }` (Screen 1 canonical structure).
- In Random Pets context (`[+]` from Screen 8), call with `family_id` omitted or `skip_pet_match: true` — `matched_pet` is always `null`.

---

### AU. `POST /families/{family_id}/pets`
Create a new pet under a family.  
**Auth:** Required (owner or parent of the family)

**Body:**
```json
{
  "name": "Snowball",
  "breed": "British Shorthair",
  "color": "grey",
  "gender": "female",
  "birthday": "2024-01-15",
  "weight_kg": 3.2,
  "avatar_url": "https://cdn.petapp.com/media/tmp_pet_avatar.jpg"
}
```

**Response `201 Created`:** Pet object with `id`, `name`, `breed`, `color`, `gender`, `age_display`, `avatar_url`.

---

### AV. `PUT /posts/draft`
Save or update the current post draft.  
**Auth:** Required

**Body:** Full post form state (same shape as `POST /posts` below, all fields optional).

**Response `200 OK`:** `{ "draft_id": "draft_abc" }`

---

### AW. `DELETE /posts/draft`
Delete the current draft (on Cancel).  
**Auth:** Required  
**Response `204 No Content`**

---

### AX. `POST /posts`
Publish the post.  
**Auth:** Required

**Body:**
```json
{
  "family_id": "fam_xyz",
  "caption": "Pudding nằm chờ mama nấu cơm 🌕",
  "privacy": "public",
  "location": {
    "city": "Hồ Chí Minh",
    "city_code": "HCM",
    "country": "Việt Nam",
    "country_code": "VN"
  },
  "media": [
    {
      "url": "https://cdn.petapp.com/media/tmp_upload_001.jpg",
      "type": "uploaded",
      "order": 1,
      "media_tag": {
        "type": "pet",
        "id": "pet_111",
        "breed": "Orange Tabby Cat"
      }
    },
    {
      "url": "https://www.youtube.com/watch?v=abc123",
      "type": "embedded",
      "order": 2,
      "media_tag": {
        "type": "random",
        "id": null,
        "breed": null
      }
    }
  ]
}
```

**Response `201 Created`:** Full post object (same shape as `GET /feed/explore` item).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `NO_MEDIA` | Post has no media items |
| `400` | `MEDIA_UNTAGGED` | One or more media items have no tag |
| `400` | `TOO_MANY_MEDIA` | More than 10 media items |
| `400` | `MULTIPLE_EMBEDDED` | More than 1 embedded URL |
| `403` | `NOT_FAMILY_MEMBER` | User is not a member of the specified family |

---

## User Flow Diagrams

### Standard Create Post

```
User opens Create Post (e.g. from fab button)
  └─> Check active family
        ├─ No active family → prompt to set in Profile Settings
        └─ Has active family → load draft (if exists) or empty form
              └─> Add media (upload or embed)
                    └─> For each uploaded media: tap AI Scan
                          └─> POST /ai/scan-media
                                ├─ Match → badge auto-set to pet name
                                ├─ Breed detected → badge shows breed (user can accept or edit)
                                └─ No detection → badge shows "Random" (user can edit)
                    └─> [Optional] edit tags manually
                    └─> Fill caption, location, privacy
                    └─> Tap Post
                          └─> POST /posts
                                ├─ 201 → DELETE /posts/draft → navigate to My Pets tab
                                └─ 400 → show validation error
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
| Media limit reached (10) | "Add Media" button disabled; counter shows "10 / 10" |
| AI scan — no pet detected | Tag: `{ type=random, breed=null }`; user can manually link to existing pet |
| AI scan — breed detected, no family pet match | Tag: `{ type=random, breed="..." }`; user can keep, link to existing pet, or create new pet |
| AI scan — match found | Tag: `{ type=pet, id=pet_xxx, breed="..." }`; user can keep, change pet, or unlink |
| User switches family | Clear all pet-type tags; prompt user to re-tag or re-scan |
| Cancel with filled form | Confirmation dialog; draft preserved on "Keep editing" |
| App crash / close mid-create | Draft auto-saved; restored on next open |
| Post button tapped with untagged media | Show validation: "All media must be tagged before posting" |
