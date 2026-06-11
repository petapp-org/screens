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
- Selected image uploaded via `SignUploadBatch (BV)` `{ items: [{ purpose: "FAMILY_AVATAR", ... }] }` (screen_4) before form submission → use `list[0].publicUrl`; no AI scan
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
| Unique | Checked in real-time via `AO. CheckFamilyTag` query |
| Prefix | Always displayed with `@` prefix (stored without `@`) |
| Editable | **Create only** — read-only (greyed out) on Update screen |

Real-time check: debounced 500ms after user stops typing.  
Show ✓ (green) if available, ✗ (red) + "Tag already taken" if not.

---

### 4. Location Fields

| Field | Required | Notes |
|-------|----------|-------|
| `city` | No | City name, e.g. `"Hồ Chí Minh"` |
| `cityCode` | No | Short code, e.g. `"HCM"` |
| `country` | No | Country name, e.g. `"Việt Nam"` |
| `countryCode` | No | Country code, e.g. `"VN"` |

- Input via searchable city/country picker (same component as Create Post location)
- Displayed on family profile as `"HCM - VN"`

---

### 5. About Field

- Multiline, free-text
- Optional
- Max 300 characters; show character counter `N/300`
- Resize handle (drag corner to expand textarea)

---

### 5. Parents Section

**Owner row** (always first, non-removable):
```
[Avatar]  [User name]  [YOU]  [OWNER]     (no action)
```

**Accepted parent row** (`status = joined`):
```
[Avatar]  [Parent name]  [PARENT]         [Remove]
```
- Tap Remove → confirmation dialog → `RemoveFamilyMember mutation (AS)`
- Row removed immediately on confirmation

**Invited parent row** (`status = invited`):
```
[Avatar]  [Invited name]  [INVITED]       [Cancel]
```
- INVITED badge highlighted (e.g. amber/orange colour)
- Tap Cancel → `CancelParentInvite mutation (AR)` → row removed

**Invite Another Parent row** (always last):
```
[+]  Invite Another Parent                [>]
```
- Tap → opens **Invite Search Modal**

---

### 6. Invite Search Modal

Bottom sheet or full-screen modal.

```
[Search input: "Search by name, @username, or phone number"]
[Results list]
  └── Each result: avatar + name + @username → tap to select → invite
[No results: "No user found"]
```

**Search behaviour:**
- Min 2 characters to trigger search
- Debounced 400ms
- `SearchUsers query (AP)`
- Results exclude: current user (owner), already-added parents, already-invited users

**Select a user:**
- Tap result → `InviteFamilyMember mutation (AQ)` `{ familyId, phoneOrEmail, role }`
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

All calls go to `POST /graphql`.

### AM. Mutation: `CreateFamily`
Create a new family.
**Auth:** Required. Returns `FAMILY_ALREADY_OWNED` error if user already owns a family.

**Operation:**
```graphql
mutation CreateFamily($input: CreateFamilyInput!) {
  createFamily(input: $input) {
    id
    name
    tag
    avatarUrl
    defaultPrivacy
    isActive
  }
}
```

**Variables:**
```json
{
  "input": {
    "name": "Thao's Family",
    "tag": "thaofam",
    "about": "Just me and a future pet or two. 🌱",
    "avatarUrl": "https://cdn.petapp.com/media/upload_xyz.jpg",
    "defaultPrivacy": "PUBLIC",
    "city": "Hồ Chí Minh",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN"
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "createFamily": {
      "id": "fam_new",
      "name": "Thao's Family",
      "tag": "thaofam",
      "avatarUrl": "https://cdn.petapp.com/families/fam_new/avatar.jpg",
      "defaultPrivacy": "PUBLIC",
      "isActive": true
    }
  }
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

### AN. Mutation: `UpdateFamily`
Update an existing family.
**Auth:** Required. Returns `403` if caller is not the owner.

**Operation:**
```graphql
mutation UpdateFamily($familyId: ID!, $input: UpdateFamilyInput!) {
  updateFamily(familyId: $familyId, input: $input) {
    id
    name
    about
    avatarUrl
    defaultPrivacy
    cityCode
    countryCode
  }
}
```

**Variables:**
```json
{
  "familyId": "fam_001",
  "input": {
    "name": "Updated Name",
    "about": "Updated description",
    "avatarUrl": "https://cdn.petapp.com/...",
    "defaultPrivacy": "FOLLOWERS"
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "updateFamily": {
      "id": "fam_001",
      "name": "Updated Name",
      "about": "Updated description",
      "avatarUrl": "https://cdn.petapp.com/...",
      "defaultPrivacy": "FOLLOWERS",
      "cityCode": "HCM",
      "countryCode": "VN"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `FORBIDDEN` | Caller is not the family owner |
| `404` | `FAMILY_NOT_FOUND` | Family ID does not exist |

---

### AO. Query: `CheckFamilyTag`
Check tag availability during Create form.
**Auth:** Not required

**Operation:**
```graphql
query CheckFamilyTag($tag: String!) {
  checkFamilyTag(tag: $tag)
}
```

**Variables:**
```json
{
  "tag": "thaofam"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "checkFamilyTag": true
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_TAG_FORMAT` | Tag contains invalid characters |

---

### AP. Query: `SearchUsers`
Search users to invite as parents.
**Auth:** Required

**Operation:**
```graphql
query SearchUsers($q: String!, $first: Int = 20, $after: String) {
  searchUsers(q: $q, first: $first, after: $after) {
    edges {
      node {
        id
        displayName
        username
        avatarUrl
      }
    }
    pageInfo {
      endCursor
      hasNextPage
    }
  }
}
```

**Variables:**
```json
{
  "q": "minhdang",
  "first": 10
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "searchUsers": {
      "edges": [
        {
          "node": {
            "id": "user_002",
            "displayName": "Minh Dang",
            "username": "minhdang",
            "avatarUrl": "https://cdn.petapp.com/users/user_002/avatar.jpg"
          }
        }
      ],
      "pageInfo": {
        "endCursor": null,
        "hasNextPage": false
      }
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `QUERY_TOO_SHORT` | Search query is fewer than 2 characters |

---

### AQ. Mutation: `InviteFamilyMember`
Send a parent invite to a user by phone or email.
**Auth:** Required (owner only)

> **Triggers notification:** when the invited user later **accepts**, an `INVITE_ACCEPTED` notification fires to the **inviter** (see screen_22 — Notifications screen). Note: the incoming invite itself is intentionally **not** surfaced as a notification (`parent invite received` is excluded — see screen_22).

**Operation:**
```graphql
mutation InviteFamilyMember($familyId: ID!, $phoneOrEmail: String!, $role: FamilyRole!) {
  inviteFamilyMember(familyId: $familyId, phoneOrEmail: $phoneOrEmail, role: $role) {
    id
    familyId
    invitedUserId
    role
    status
    createdAt
  }
}
```

**Variables:**
```json
{
  "familyId": "fam_001",
  "phoneOrEmail": "minh@example.com",
  "role": "PARENT"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "inviteFamilyMember": {
      "id": "inv_001",
      "familyId": "fam_001",
      "invitedUserId": "user_002",
      "role": "PARENT",
      "status": "INVITED",
      "createdAt": "2026-01-01T00:00:00Z"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `FORBIDDEN` | Caller is not the family owner |
| `409` | `ALREADY_MEMBER` | User is already a parent or has a pending invite |
| `404` | `USER_NOT_FOUND` | Target user does not exist |

**Parent `status` enum:**

| Value | Meaning |
|-------|---------|
| `INVITED` | Invite sent, awaiting acceptance |
| `JOINED` | User accepted and is an active parent |

> Rejected invites are removed from the list entirely — the owner must re-invite the user if needed.

---

### AR. Mutation: `CancelParentInvite`
Cancel a pending parent invite.
**Auth:** Required (owner only)

**Operation:**
```graphql
mutation CancelParentInvite($familyId: ID!, $userId: ID!) {
  cancelParentInvite(familyId: $familyId, userId: $userId) {
    success
  }
}
```

**Variables:**
```json
{
  "familyId": "fam_001",
  "userId": "user_002"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "cancelParentInvite": {
      "success": true
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `FORBIDDEN` | Caller is not the family owner |
| `404` | `INVITE_NOT_FOUND` | No pending invite exists for this user |

---

### AS. Mutation: `RemoveFamilyMember`
Remove an accepted parent from the family.
**Auth:** Required (owner only). Cannot remove self (owner).

**Side effects for the removed user (see screen_10):** They lose all access to this family's received chat threads **and** the entire chat history for this family. If this family was the removed user's **active** family, the server **auto-switches** their active family to another family they belong to (or unsets it if none) — this applies to both the Chats inbox scope and the action/context everywhere.

**Operation:**
```graphql
mutation RemoveFamilyMember($familyId: ID!, $userId: ID!) {
  removeFamilyMember(familyId: $familyId, userId: $userId) {
    id
    members {
      userId
      role
    }
  }
}
```

**Variables:**
```json
{
  "familyId": "fam_001",
  "userId": "user_003"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "removeFamilyMember": {
      "id": "fam_001",
      "members": [
        { "userId": "user_001", "role": "OWNER" }
      ]
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `FORBIDDEN` | Caller is not the family owner |
| `403` | `CANNOT_REMOVE_OWNER` | Attempt to remove the owner themselves |
| `404` | `MEMBER_NOT_FOUND` | User is not a member of this family |

---

## User Flow Diagrams

### Create Family

```
User taps "Create Family Page" in Profile Settings
  └─> Navigate to Create Family screen
        └─> Fill in name, tag (real-time check), about, privacy
              └─> Optionally upload avatar (media upload endpoint)
                    └─> Optionally invite parents → SearchUsers query (AP) → InviteFamilyMember mutation (AQ)
                          └─> Tap "Create Family"
                                └─> CreateFamily mutation (AM)
                                      ├─ TAG_TAKEN → show error on tag field
                                      └─ 201 → navigate to new Family Posts screen
                                                (new family auto-set as active)
```

### Invite Parent

```
Tap "Invite Another Parent"
  └─> Open Invite Search Modal
        └─> Type ≥2 chars → SearchUsers query (AP)
              ├─ Results shown → tap user → InviteFamilyMember mutation (AQ)
              │     └─ Close modal → INVITED row added to Parents list
              └─ No results → "No user found" — cannot invite
```

### Update Family

```
Owner taps Edit in Family Posts screen
  └─> Navigate to Update Family screen (fields pre-filled)
        └─> Edit fields
              └─> Tap "Save Changes" → UpdateFamily mutation (AN)
                    └─> success → navigate back, family info updated
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
