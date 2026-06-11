# Screen 5: Profile Settings ("Me")

## Overview

The user's personal hub — accessible from the **top-right header avatar** (visible on the Explore screen and other screens).  
Requires login. Not related to the "More" tab in the bottom navigation.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Me"

[Scrollable area]
  ├── User Header
  │     ├── Avatar (tappable → Edit Profile)
  │     ├── Display name
  │     └── @username
  │
  ├── FAMILY PAGES section
  │     ├── [Create Family Page row] — shown only if user has no owned family
  │     │     OR
  │     │     [Owned family row] — shown if user already owns a family
  │     └── [Joined family rows] — one per family the user is a parent of (not owner)
  │           Each row: family avatar · family name · @tag · [Active toggle]
  │
  ├── ACTIVITY section
  │     ├── Loves  [>]
  │     └── Following  [>]
  │
  └── SETTINGS section
        ├── Account Settings  [>]
        ├── Language  [>]
        └── [Log out button]
```

---

## Components

### 1. User Header

| Field | Description |
|-------|-------------|
| `avatarUrl` | User avatar — tappable, opens Edit Profile screen |
| `name` | Display name |
| `username` | Unique username, displayed as `@username` |

---

### 2. Family Pages Section

**Rule: only 1 family can be active at any time.**  
Active family is used for: receiving push notifications directed at the family; **scoping the Chats inbox in Messages (screen_10)** — which family-received threads are visible and which messages count as unread (only messages sent *after* the family became active); and as the default family when creating a new post.

**Create Family Page row** (shown when user has no owned family):
- Icon: `+`
- Label: "Create Family Page"
- Tap → navigate to Create Family screen

**Owned family row** (shown when user already owns a family):
- Family avatar + family name + `@tag` + `OWNER` badge
- Active toggle (toggle on = this family is active)
- Tap on row (not toggle) → navigate to Family Posts screen for that family

**Joined family rows** (one per family where user is a parent, not owner):
- Family avatar + family name + `@tag` + `PARENT` badge
- Active toggle
- Tap on row → navigate to Family Posts screen

**Active toggle behaviour:**
- Toggling ON a family → sets it as active; all other families toggle OFF automatically
- Only 1 can be ON at a time across owned + joined families
- Creating a new family → auto-activates it (all others deactivated)
- `MarkFamilyPrimary mutation (AG)` `{ familyId }` → `200 OK`

---

### 3. Activity — Loves

Navigates to **Loved Posts screen**:
- Only accessible by the current user (private — no way to view another user's loved posts)
- No filter tabs (unlike Explore)
- No Suggested Families widget
- Sorted by most recently loved first
- 10 posts per page, infinite scroll
- API: `MyLovedPosts query (AH)`

**Post card:**
- Same post card format as Explore — all canonical tap interactions apply (see `screen_1_home_explore.md` → Post Card): tap **family name** → Family Posts, tap **author name** → User Posts, tap **pet badge** → Pet Posts
- Tap **Love button** → un-love; post is removed from the list immediately (optimistic); on API error revert and show error toast

**Empty state** (user has not loved any posts):
- Message: "You haven't loved any posts yet"

---

### 4. Activity — Following

Navigates to **Following screen**:
- Sorted by most recently followed first
- 20 families per page; **"Load more"** button at bottom — each tap loads 20 more
- API: `MyFollowing query (AI)`

**Each row:**
- Family avatar + family name + `@tag`
- Tap anywhere on the row (except action buttons) → **Family Posts screen**

> followersCount trả số thô; client tự format "3.6k" theo locale (bỏ followerCountDisplay — i18n client-side).
- `NORMAL` family → **Following button** only
- `CHARITY` family → **Following button** + **Donate button**
  - Tap Donate → **opens a chat** with that charity (pre-filled VN support message) — interim, no wallet; canonical in `screen_3`. Login required; hidden for members of that charity.

**Unfollow:**
- Tap **Following button** → unfollow
  - Optimistic: remove row immediately
  - Show toast: "Unfollowed [Family name]" + **Undo** button (5s window)
  - Tap Undo → `FollowFamily mutation (D)` → re-add row
  - No Undo tap → `UnfollowFamily mutation (E)` confirmed

**Empty state** (user follows no one):
- Message: "You're not following anyone yet"
- Show **Suggested Families widget** (same component as Explore feed — see `screen_1_home_explore.md` → Suggested Families Widget)

---

### 5. Settings — Account Settings

Sub-screen containing:

```
PROFILE
  Edit Profile  [>]       ← avatar, name (username is read-only after setup)
  Phone & Email  [>]      ← view/update phone number and linked email

PREFERENCES
  Push Notifications  [toggle]
  Language  [>]           ← language picker

[Log out]                 ← red text button, confirmation dialog before logging out

DANGER ZONE
  Delete Account  [>]     ← navigates to separate Delete Account screen (1 level deeper)
```

**Edit Profile:**
- Editable: avatar, name
- Read-only: `username` (shown greyed out with note "Username cannot be changed")
- Changing the avatar: upload via `SignUploadBatch (BV)` `{ items: [{ purpose: "AVATAR", ... }] }` (screen_4) → use `list[0].publicUrl`; no AI scan
- `UpdateMyProfile mutation (AK)` `{ displayName, avatarUrl }`

**Phone & Email:**
- Shows current phone number (masked, e.g. `+84 *** *** 567`)
- Shows linked email (from OAuth)
- API: `MyContactInfo query (AJ)`

**Push Notifications:**
- Toggle stored locally + synced to server
- `UpdateUserPreferences mutation` `{ pushEnabled: true|false }`

**Language:**
- Picker for app display language
- Stored locally (client preference); no server sync needed

**Log out:**
- Tap → confirmation dialog: "Are you sure you want to log out?"
- Confirm → clear local tokens → redirect to Register/Login screen
- `Logout mutation (AL)` `{ sessionId, refreshToken }` (invalidates refresh token server-side)

**Delete Account screen** (1 level deeper — tap "Delete Account [>]" to navigate here):
- Explains consequences: posts, pets, and family data will be removed
- Explains grace period: "Your account will be permanently deleted after 30 days. Log in again within 30 days to cancel deletion."
- Single **"Delete My Account"** button (red, destructive style)
- Tap → confirmation dialog: "Are you sure? This starts a 30-day deletion period." → Confirm → `RequestAccountDeletion mutation`
- After confirm: log out immediately → redirect to Register/Login screen
- Server sets `scheduledAt = now + 30 days` and `status = PENDING` on the deletion request

**Reactivation (within 30 days):**
- User logs in via OAuth → server detects a pending deletion request (`status = PENDING`) → clears it (sets `status` back to cancelled / removes the record) → account fully restored
- Show toast on login: "Welcome back! Your account deletion has been cancelled."

---

## API Endpoints Required

All calls go to `POST /graphql`.

---

### Query: `Me` *(shared — no assigned letter ID)*
Fetch current user profile (for the Me screen header), including all family memberships.  
**Auth:** Required

**Operation:**
```graphql
query Me {
  me {
    id
    displayName
    username
    avatarUrl
    families {
      id
      name
      tag
      avatarUrl
      familyType
      isPrimary
      role
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
    "me": {
      "id": "user_001",
      "displayName": "Minh Dang",
      "username": "minhdang",
      "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg",
      "families": [
        {
          "id": "fam_xyz",
          "name": "Minh's Family",
          "tag": "minhfamily",
          "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
          "familyType": "NORMAL",
          "isPrimary": true,
          "role": "OWNER"
        },
        {
          "id": "fam_abc",
          "name": "Cecilia's Family",
          "tag": "ceciliafam",
          "avatarUrl": "https://cdn.petapp.com/families/fam_abc/avatar.jpg",
          "familyType": "NORMAL",
          "isPrimary": false,
          "role": "MEMBER"
        }
      ]
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### AG. Mutation: `MarkFamilyPrimary`
Set the primary (active) family for the current user. All other families are deactivated automatically.  
**Auth:** Required

**Side effects (see screen_10):** Switching the primary family re-scopes the Chats inbox — threads received by the previous active family disappear, threads received by the new active family appear (DMs and own-sent threads are unaffected). The server records the activation moment; a family-received message counts as **unread only if** `sentAt > activation time`, so messages that arrived while the family was inactive are not marked unread.

**Operation:**
```graphql
mutation MarkFamilyPrimary($familyId: ID!) {
  markFamilyPrimary(familyId: $familyId) {
    id
    isPrimary
  }
}
```

**Variables:**
```json
{
  "familyId": "fam_xyz"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "markFamilyPrimary": {
      "id": "fam_xyz",
      "isPrimary": true
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `FAMILY_NOT_FOUND` | Family does not exist |
| `NOT_A_MEMBER` | Caller is not a member of the specified family |

---

### AH. Query: `MyLovedPosts`
Paginated list of posts the current user has loved, sorted by love date descending.  
**Auth:** Required

**Operation:**
```graphql
query MyLovedPosts($after: String) {
  myLovedPosts(first: 20, after: $after) {
    edges {
      cursor
      node {
        id
        family { id name avatarUrl familyType }
        author { id displayName avatarUrl }
        pets { id name avatarUrl }
        body
        location { city cityCode country countryCode }
        media { id url mimeType width height durationSeconds }
        loveCount
        commentsCount
        isLoved
        createdAt
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Variables:**
```json
{
  "after": null
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "myLovedPosts": {
      "edges": [
        {
          "cursor": "cursor_token_here",
          "node": {
            "id": "post_001",
            "family": { "id": "fam_xyz", "name": "Minh's Family", "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg", "familyType": "NORMAL" },
            "author": { "id": "user_001", "displayName": "Minh Tuan", "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg" },
            "pets": [ { "id": "pet_111", "name": "Bụi", "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg" } ],
            "body": "Bụi nằm chờ mama nấu cơm 🌕",
            "location": { "city": "Hồ Chí Minh", "cityCode": "HCM", "country": "Việt Nam", "countryCode": "VN" },
            "media": [ { "id": "media_001", "url": "https://cdn.petapp.com/media/001.jpg", "mimeType": "image/jpeg", "width": 1080, "height": 1080, "durationSeconds": null } ],
            "loveCount": 24,
            "commentsCount": 3,
            "isLoved": true,
            "createdAt": "2026-06-06T08:00:00Z"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": true,
        "endCursor": "cursor_token_here"
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### AI. Query: `MyFollowing`
Paginated list of families the current user follows.  
**Auth:** Required

> ✅ **Resolved (petapp-be#921):** backend now exposes `myFollowingFamilies(first, after): FamilyConnection!` (Relay). `Family` carries no `city`/`country` fields, so those are dropped from the selection.

**Operation:**
```graphql
query MyFollowing($first: Int! = 20, $after: String) {
  myFollowingFamilies(first: $first, after: $after) {
    edges {
      cursor
      node {
        id
        name
        tag
        avatarUrl
        familyType
        social {
          followersCount
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Variables:**
```json
{
  "first": 20,
  "after": null
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "myFollowingFamilies": {
      "edges": [
        {
          "cursor": "eyJpZCI6ImZhbV94eXoifQ==",
          "node": {
            "id": "fam_xyz",
            "name": "Daily Cats",
            "tag": "dailycats",
            "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
            "familyType": "NORMAL",
            "social": {
              "followersCount": 3840
            }
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": false,
        "endCursor": "eyJpZCI6ImZhbV94eXoifQ=="
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### AJ. Query: `MyContactInfo`
Fetch the current user's phone number and linked email address.  
**Auth:** Required

**Operation:**
```graphql
query MyContactInfo {
  myContactInfo {
    phone
    email
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
    "myContactInfo": {
      "phone": "+84901234567",
      "email": "minh@example.com"
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### AK. Mutation: `UpdateMyProfile`
Update the current user's editable profile fields. All fields are optional; only provided fields are updated.  
**Auth:** Required

**Operation:**
```graphql
mutation UpdateMyProfile($displayName: String, $avatarUrl: String) {
  updateMyProfile(displayName: $displayName, avatarUrl: $avatarUrl) {
    id
    displayName
    avatarUrl
  }
}
```

**Variables:**
```json
{
  "displayName": "Minh Dang",
  "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "updateMyProfile": {
      "id": "user_001",
      "displayName": "Minh Dang",
      "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `DISPLAY_NAME_TOO_LONG` | `displayName` exceeds 50 characters |

> The push-notification toggle uses a separate op, `updateUserPreferences`:

```graphql
mutation UpdateUserPreferences($pushEnabled: Boolean) {
  updateUserPreferences(pushEnabled: $pushEnabled) {
    pushEnabled
  }
}
```

(`pushEnabled` replaces the old `pushNotificationsEnabled` field.)

---

### AL. Mutation: `Logout`
Invalidate the current user's refresh token server-side and end the session.  
**Auth:** Required

**Operation:**
```graphql
mutation Logout($sessionId: String!, $refreshToken: String!) {
  logout(sessionId: $sessionId, refreshToken: $refreshToken) {
    revoked
  }
}
```

**Variables:**
```json
{
  "sessionId": "sess_abc",
  "refreshToken": "eyJ..."
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "logout": {
      "revoked": true
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### Mutation: `RequestAccountDeletion` *(no assigned letter ID)*

Schedule the current user's account for deletion after a 30-day grace period.  
**Auth:** Required

**Operation:**
```graphql
mutation RequestAccountDeletion {
  requestAccountDeletion {
    status
    scheduledAt
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
    "requestAccountDeletion": {
      "status": "PENDING",
      "scheduledAt": "2026-07-07T12:00:00Z"
    }
  }
}
```

**Side effects:**
- Sets `scheduledAt = now + 30 days` and `status = PENDING` on the account deletion request
- Caller is immediately logged out client-side after receiving response
- User logging back in within 30 days via OAuth cancels the deletion (`status` cleared back to cancelled); show toast "Welcome back! Your account deletion has been cancelled."

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `DELETION_ALREADY_SCHEDULED` | Account already has a pending deletion request |

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| User has no owned family | Show "Create Family Page" row; no toggle shown |
| User owns a family | Replace "Create Family Page" with owned family row (OWNER badge) |
| User tries to create 2nd family | `CreateFamily mutation` returns `FAMILY_ALREADY_OWNED`; show error |
| Toggle ON a family | All other toggles turn OFF immediately (optimistic); `MarkFamilyPrimary mutation (AG)` |
| Unfollow with Undo | Row removed immediately; toast with 5s Undo window; confirmed on timeout or close |
| No loved posts | Empty state: "You haven't loved any posts yet" |
| No followed families | Empty state: "You're not following any families yet" |
| Log out | Clears access + refresh tokens locally; server invalidates refresh token |
