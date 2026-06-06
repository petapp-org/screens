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
  │     └── @tag
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
| `avatar_url` | User avatar — tappable, opens Edit Profile screen |
| `name` | Display name |
| `tag` | Unique tag, displayed as `@tag` |

---

### 2. Family Pages Section

**Rule: only 1 family can be active at any time.**  
Active family is used for: receiving push notifications directed at the family, and as the default family when creating a new post.

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
- `PATCH /users/me/active-family` `{ family_id }` → `200 OK`

---

### 3. Activity — Loves

Navigates to **Loved Posts screen**:
- List of posts the user has loved
- Sorted `created_at` desc (latest loved first)
- 10 posts per page, infinite scroll
- Same post card format as Explore
- API: `GET /users/me/loved-posts?cursor=&limit=10`

---

### 4. Activity — Following

Navigates to **Following screen**:
- List of families the user follows
- Each row: family avatar + family name + `@tag` + city/country + **Following button**
- Tap **Following button** → unfollow
  - Optimistic: remove row immediately
  - Show toast: "Unfollowed [Family name]" + **Undo** button (5s window)
  - Tap Undo → `POST /families/{id}/follow` → re-add row
  - No Undo tap → `DELETE /families/{id}/follow` confirmed
- API: `GET /users/me/following?cursor=&limit=20`

---

### 5. Settings — Account Settings

Sub-screen containing:

```
PROFILE
  Edit Profile  [>]       ← avatar, name (tag is read-only after setup)
  Phone & Email  [>]      ← view/update phone number and linked email

PREFERENCES
  Push Notifications  [toggle]
  Language  [>]           ← language picker

[Log out]                 ← red text button, confirmation dialog before logging out
```

**Edit Profile:**
- Editable: avatar, name
- Read-only: `tag` (shown greyed out with note "Tag cannot be changed")
- `PATCH /users/me` `{ name, avatar_url }`

**Phone & Email:**
- Shows current phone number (masked, e.g. `+84 *** *** 567`)
- Shows linked email (from OAuth or added manually)
- Option to add/change email
- API: `GET /users/me/contact-info`, `PATCH /users/me/contact-info`

**Push Notifications:**
- Toggle stored locally + synced to server
- `PATCH /users/me/preferences` `{ push_notifications_enabled: true|false }`

**Language:**
- Picker for app display language
- Stored locally (client preference); no server sync needed

**Log out:**
- Tap → confirmation dialog: "Are you sure you want to log out?"
- Confirm → clear local tokens → redirect to Register/Login screen
- `POST /auth/logout` `{ refresh_token }` (invalidates refresh token server-side)

---

## API Endpoints Required

All calls go to `POST /graphql`.

---

### AF. Query: `Me`
Fetch current user profile (for the Me screen header), including all family memberships.  
**Auth:** Required

**Operation:**
```graphql
query Me {
  me {
    id
    displayName
    tag
    avatarUrl
    families {
      id
      name
      tag
      avatarUrl
      type
      isActive
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
      "tag": "minhdang",
      "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg",
      "families": [
        {
          "id": "fam_xyz",
          "name": "Minh's Family",
          "tag": "minhfamily",
          "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
          "type": "STANDARD",
          "isActive": true,
          "role": "OWNER"
        },
        {
          "id": "fam_abc",
          "name": "Cecilia's Family",
          "tag": "ceciliafam",
          "avatarUrl": "https://cdn.petapp.com/families/fam_abc/avatar.jpg",
          "type": "STANDARD",
          "isActive": false,
          "role": "PARENT"
        }
      ]
    }
  }
}
```

---

### AG. Mutation: `SetActiveFamily`
Set the active family for the current user. All other families are deactivated automatically.  
**Auth:** Required

**Operation:**
```graphql
mutation SetActiveFamily($input: SetActiveFamilyInput!) {
  setActiveFamily(input: $input) {
    activeFamilyId
  }
}
```

**Variables:**
```json
{
  "input": {
    "familyId": "fam_xyz"
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "setActiveFamily": {
      "activeFamilyId": "fam_xyz"
    }
  }
}
```

---

### AH. Query: `MyLovedPosts`
Paginated list of posts the current user has loved, sorted by love date descending.  
**Auth:** Required

**Operation:**
```graphql
query MyLovedPosts($cursor: String, $limit: Int) {
  myLovedPosts(cursor: $cursor, limit: $limit) {
    posts {
      ...Post
    }
    nextCursor
    hasMore
  }
}
```

**Variables:**
```json
{
  "cursor": null,
  "limit": 10
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "myLovedPosts": {
      "posts": [ "...same Post shape as Screen 1..." ],
      "nextCursor": "cursor_token_here",
      "hasMore": true
    }
  }
}
```

---

### AI. Query: `MyFollowing`
Paginated list of families the current user follows.  
**Auth:** Required

**Operation:**
```graphql
query MyFollowing($cursor: String, $limit: Int) {
  myFollowing(cursor: $cursor, limit: $limit) {
    families {
      id
      name
      tag
      avatarUrl
      city
      country
      followerCount
    }
    nextCursor
    hasMore
  }
}
```

**Variables:**
```json
{
  "cursor": null,
  "limit": 20
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "myFollowing": {
      "families": [
        {
          "id": "fam_xyz",
          "name": "Daily Cats",
          "tag": "dailycats",
          "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
          "city": "Internet",
          "country": "",
          "followerCount": 3840
        }
      ],
      "nextCursor": null,
      "hasMore": false
    }
  }
}
```

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

---

### AK. Mutation: `UpdateMe`
Update the current user's editable profile fields and/or preferences. All fields are optional; only provided fields are updated.  
**Auth:** Required

**Operation:**
```graphql
mutation UpdateMe($input: UpdateMeInput!) {
  updateMe(input: $input) {
    id
    displayName
    avatarUrl
    pushNotificationsEnabled
  }
}
```

**Variables:**
```json
{
  "input": {
    "displayName": "Minh Dang",
    "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg",
    "pushNotificationsEnabled": true
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "updateMe": {
      "id": "user_001",
      "displayName": "Minh Dang",
      "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg",
      "pushNotificationsEnabled": true
    }
  }
}
```

---

### AL. Mutation: `Logout`
Invalidate the current user's refresh token server-side and end the session.  
**Auth:** Required

**Operation:**
```graphql
mutation Logout($input: LogoutInput!) {
  logout(input: $input) {
    success
  }
}
```

**Variables:**
```json
{
  "input": {
    "refreshToken": "eyJ..."
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "logout": {
      "success": true
    }
  }
}
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| User has no owned family | Show "Create Family Page" row; no toggle shown |
| User owns a family | Replace "Create Family Page" with owned family row (OWNER badge) |
| User tries to create 2nd family | `POST /families` returns `403 FAMILY_ALREADY_OWNED`; show error |
| Toggle ON a family | All other toggles turn OFF immediately (optimistic); `PATCH /users/me/active-family` |
| Unfollow with Undo | Row removed immediately; toast with 5s Undo window; confirmed on timeout or close |
| No loved posts | Empty state: "You haven't loved any posts yet" |
| No followed families | Empty state: "You're not following any families yet" |
| Log out | Clears access + refresh tokens locally; server invalidates refresh token |
