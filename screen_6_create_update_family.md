# Screen 6: Create / Update Family

## Overview

Form screen to create a new family page or edit an existing one.  
- **Create**: accessed from "Create Family Page" row in Profile Settings. Each user can own at most 1 family.  
- **Update**: accessed from family settings within the family page (owner only).  
Requires login. Only the family owner can edit.

---

## UI Layout

```
[Header]
  Left: Back / Cancel
  Center: "Create Family" | "Edit Family"

[Scrollable area]
  ├── Avatar picker
  │     └── Circular avatar + camera icon overlay → opens image picker
  │
  ├── Name field
  │     └── Text input (required)
  │
  ├── Tag field
  │     └── "@" prefix + text input (required, unique)
  │         Real-time availability indicator: ✓ available / ✗ taken
  │         [Edit mode: read-only — tag cannot be changed after creation]
  │
  ├── About field
  │     └── Multiline text input (optional)
  │
  ├── PARENTS section  [count badge]  [expand/collapse ▾]
  │     ├── [Owner row] — current user, YOU badge, "Creator" label, cannot be removed
  │     ├── [Accepted parent rows] — PARENT badge + Remove button
  │     ├── [Invited parent rows] — INVITED badge (highlighted) + Cancel button
  │     └── [Invite Another Parent row] — tap → opens invite search modal
  │
  ├── PRIVACY section  [current value]  [expand/collapse ▾]
  │     └── Dropdown: Public | Followers only | Family only
  │           Description text below selected option
  │
  └── (spacer for fixed button)

[Fixed bottom button]
  "Create Family" | "Save Changes"
```

---

## Components

### 1. Avatar Picker

- Tap → opens device image picker
- Selected image uploaded via `POST /media/upload` before form submission
- Shows upload progress indicator on the avatar
- On Create: optional (default avatar used if skipped)
- On Update: always shows current avatar

---

### 2. Name Field

| Rule | Detail |
|------|--------|
| Required | Cannot submit without a name |
| Max length | 50 characters |
| Character counter | Show `N/50` below field |

---

### 3. Tag Field

| Rule | Detail |
|------|--------|
| Required | Cannot submit without a tag |
| Format | Letters, numbers, underscores only; no spaces; max 30 chars |
| Unique | Checked in real-time against `GET /families/check-tag?tag=...` |
| Prefix | Always displayed with `@` prefix (stored without `@`) |
| Editable | **Create only** — read-only (greyed out) on Update screen |

Real-time check: debounced 500ms after user stops typing.  
Show ✓ (green) if available, ✗ (red) + "Tag already taken" if not.

---

### 4. About Field

- Multiline, free-text
- Optional
- Max 300 characters; show character counter `N/300`
- Resize handle (drag corner to expand textarea)

---

### 5. Parents Section

**Owner row** (always first, non-removable):
```
[Avatar]  [User name]  [YOU]              Creator
```

**Accepted parent row** (`status = joined`):
```
[Avatar]  [Parent name]  [PARENT]         [Remove]
```
- Tap Remove → confirmation dialog → `DELETE /families/{id}/parents/{user_id}`
- Row removed immediately on confirmation

**Invited parent row** (`status = invited`):
```
[Avatar]  [Invited name]  [INVITED]       [Cancel]
```
- INVITED badge highlighted (e.g. amber/orange colour)
- Tap Cancel → `DELETE /families/{id}/invites/{user_id}` → row removed

**Invite Another Parent row** (always last):
```
[+]  Invite Another Parent                [>]
```
- Tap → opens **Invite Search Modal**

---

### 6. Invite Search Modal

Bottom sheet or full-screen modal.

```
[Search input: "Search by name, @tag, or phone number"]
[Results list]
  └── Each result: avatar + name + @tag → tap to select → invite
[No results: "No user found"]
```

**Search behaviour:**
- Min 2 characters to trigger search
- Debounced 400ms
- `GET /users/search?q=minhdang&limit=10`
- Results exclude: current user (owner), already-added parents, already-invited users

**Select a user:**
- Tap result → `POST /families/{id}/invites` `{ user_id }`
- Close modal → new row appears in Parents list with INVITED status

**No match:**
- Show "No user found" — cannot invite if no match returned

---

### 7. Privacy Section

Dropdown with 3 options. Determines the **default privacy** applied to new posts created under this family. Each post retains its own privacy setting and can be changed individually after creation.

| Option | Value | Description shown in UI |
|--------|-------|--------------------------|
| Public | `public` | Anyone can view your posts |
| Followers only | `followers` | Only your followers can view your posts |
| Family only | `private` | Only family members can view your posts |

- Default on Create: `public`
- Selecting an option updates the displayed description text below

---

### 8. Fixed Bottom Button

- **Create mode:** "Create Family" — disabled until Name + Tag are filled and tag is available
- **Update mode:** "Save Changes" — disabled if no changes made
- On tap → validate form → submit API call

---

## API Endpoints Required

### AM. `POST /families`
Create a new family.  
**Auth:** Required. Returns `403 FAMILY_ALREADY_OWNED` if user already owns a family.

**Body:**
```json
{
  "name": "Thao's Family",
  "tag": "thaofam",
  "about": "Just me and a future pet or two. 🌱",
  "avatar_url": "https://cdn.petapp.com/media/upload_xyz.jpg",
  "default_privacy": "public"
}
```

**Response `201 Created`:**
```json
{
  "id": "fam_new",
  "name": "Thao's Family",
  "tag": "thaofam",
  "avatar_url": "https://cdn.petapp.com/families/fam_new/avatar.jpg",
  "default_privacy": "public",
  "is_active": true
}
```

Note: new family is automatically set as the user's active family.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `FAMILY_ALREADY_OWNED` | User already owns a family |
| `409` | `TAG_TAKEN` | Tag already in use by another family |
| `400` | `INVALID_TAG_FORMAT` | Tag contains invalid characters |

---

### AN. `PATCH /families/{family_id}`
Update an existing family.  
**Auth:** Required. Returns `403` if caller is not the owner.

**Body (partial update):**
```json
{
  "name": "Updated Name",
  "about": "Updated description",
  "avatar_url": "https://cdn.petapp.com/...",
  "default_privacy": "followers"
}
```

**Response `200 OK`:** Updated family object.

---

### AO. `GET /families/check-tag`
Check tag availability during Create form.  
**Auth:** Not required  
**Query:** `?tag=thaofam`  
**Response `200 OK`:** `{ "available": true }`

---

### AP. `GET /users/search`
Search users to invite as parents.  
**Auth:** Required  
**Query:** `?q=minhdang&limit=10`  
**Response `200 OK`:**
```json
{
  "users": [
    {
      "id": "user_002",
      "name": "Minh Dang",
      "tag": "minhdang",
      "avatar_url": "https://cdn.petapp.com/users/user_002/avatar.jpg"
    }
  ]
}
```

---

### AQ. `POST /families/{family_id}/invites`
Send a parent invite to a user.  
**Auth:** Required (owner only)  
**Body:** `{ "user_id": "user_002" }`  
**Response `201 Created`:** `{ "status": "invited", "user_id": "user_002" }`

---

### AR. `DELETE /families/{family_id}/invites/{user_id}`
Cancel a pending invite.  
**Auth:** Required (owner only)  
**Response `204 No Content`**

---

### AS. `DELETE /families/{family_id}/parents/{user_id}`
Remove an accepted parent from the family.  
**Auth:** Required (owner only). Cannot remove self (owner).  
**Response `204 No Content`**

---

## User Flow Diagrams

### Create Family

```
User taps "Create Family Page" in Profile Settings
  └─> Navigate to Create Family screen
        └─> Fill in name, tag (real-time check), about, privacy
              └─> Optionally upload avatar → POST /media/upload
                    └─> Optionally invite parents → GET /users/search → POST /families/{id}/invites
                          └─> Tap "Create Family"
                                └─> POST /families
                                      ├─ 409 TAG_TAKEN → show error on tag field
                                      └─ 201 → navigate to new Family Posts screen
                                                (new family auto-set as active)
```

### Invite Parent

```
Tap "Invite Another Parent"
  └─> Open Invite Search Modal
        └─> Type ≥2 chars → GET /users/search?q=...
              ├─ Results shown → tap user → POST /families/{id}/invites
              │     └─ Close modal → INVITED row added to Parents list
              └─ No results → "No user found" — cannot invite
```

### Update Family

```
Owner taps Edit in Family Posts screen
  └─> Navigate to Update Family screen (fields pre-filled)
        └─> Edit fields
              └─> Tap "Save Changes" → PATCH /families/{id}
                    └─> 200 → navigate back, family info updated
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| User already owns a family, taps Create | `403 FAMILY_ALREADY_OWNED`; show error — should not normally reach this screen |
| Tag taken on submit | `409 TAG_TAKEN`; highlight tag field with error |
| Tag already set (Update mode) | Tag field is read-only, greyed out |
| No avatar uploaded | Default avatar used; no error |
| Invite search — user already in family or invited | Excluded from results (server filters) |
| Remove owner row | Not possible; Remove button not shown for owner row |
| Cancel all invites then submit | Allowed — family created with owner only |
| Privacy change on Update | Only affects new posts from that point forward; existing posts keep their own privacy |
