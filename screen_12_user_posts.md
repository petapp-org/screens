# Screen 12: User Posts

## Overview

Full post feed for a single user — shows all posts authored by that user that the viewer is allowed to see.  
Navigated to from: tapping **author name** on any post card (Explore, Post Detail, Family Posts, Pet Posts, Random Pet Posts, Loved Posts); tapping a **parent name** in the Parents bottom sheet (Family Posts, My Pets).  
Accessible without login — unauthenticated users see only `public` posts.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: @tag (static)

[Scrollable area]
  ├── User Info Card
  │     ├── [Avatar]
  │     ├── Display name
  │     └── @tag
  │
  └── Posts section
        ├── "POSTS"  [Grid view icon]  [List view icon]
        └── Post list (list view) or media grid (grid view)
```

---

## Components

### 1. User Info Card

| Field | Display |
|-------|---------|
| `avatarUrl` | User avatar (circular) |
| `displayName` | User's display name (bold) |
| `tag` | `@tag` in muted text below name |

No follow button — following is per-family, not per-user.  
No message button — messaging is per-family thread, not user-to-user.

---

### 2. Posts List

- Same post card format as Explore — all canonical tap interactions apply (see `screen_1_home_explore.md` → Post Card):
  - Tap **family name** → Family Posts screen
  - Tap **author name** → (this screen, same user — no navigation)
  - Tap **pet badge** → Pet Posts screen
  - Tap **post body / media** → Post Detail screen
- Sorted `createdAt` desc (newest first)
- 10 posts per page, infinite scroll
- List/grid view toggle (same as Family Posts screen)
- Grid view: 3-column thumbnail grid; tap cell → Post Detail screen

**Privacy rules:**
- Unauthenticated viewer → `public` posts only
- Authenticated, following the post's family → `public` + `followers`
- Authenticated, family member → `public` + `followers` + `private`
- Server enforces — client receives only visible posts

**Empty state:** Not applicable — screen is only reachable by tapping a post card authored by this user, so at least 1 post is always visible to the viewer.

---

## API Endpoints Required

Reuses endpoint defined in `screen_3_family_posts.md`.

### V. Query: `UserPosts`

See full definition in `screen_3_family_posts.md` → **Section: V. Query: UserPosts**.

**Variables:** `{ "userId": "<user_id>", "limit": 10 }`

---

## User Flow Diagrams

### Open User Posts screen

```
User taps author name on a post card
  └─> UserPosts query (V) { userId, limit: 10 }
        ├─ USER_NOT_FOUND → show "User not found" error state
        └─ 200 → render user info card + posts list
```

### Infinite Scroll

```
User scrolls to bottom
  └─> UserPosts query (V) { userId, cursor: <next_cursor>, limit: 10 }
        └─> Append new posts
              └─> hasMore=false → show "No more posts" state
```

### Switch to Grid View

```
User taps grid view icon
  └─> Re-render existing loaded posts as 3-column grid (no new API call)
        └─> Scroll to load more → UserPosts query (V) { userId, cursor: <next_cursor> }
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| User has no visible posts | Empty state: "No posts yet" |
| Viewer taps their own name | Same screen renders normally (shows own posts) |
| Post privacy mixed (some public, some private) | Server returns only posts visible to this viewer; gaps in count are expected |
| User deleted account | `USER_NOT_FOUND` → show "User not found" error state with Back button |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Post count in header | Not shown |
| 2 | Follow/message buttons | Not shown — following is per-family, messaging is per-family thread |
| 3 | Default post view | List view |
| 4 | Empty state reachability | Screen is only opened via tap on an existing post, so at least 1 post is always visible |
