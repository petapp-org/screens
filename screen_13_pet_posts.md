# Screen 13: Pet Posts

## Overview

Full post feed for a single named pet — shows all posts where that pet is tagged.  
Navigated to from: tapping **pet badge** on any post card media (Explore, Post Detail, Family Posts, User Posts, Random Pet Posts, Loved Posts); tapping a **pet row** in the Family Posts pets list.  
Accessible without login — unauthenticated users see only `public` posts.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: Pet name (static)

[Scrollable area]
  ├── Pet Info Card
  │     ├── [Avatar]
  │     ├── Pet name
  │     ├── Breed  (or species if breed is null)
  │     └── Species  (shown below breed, muted)
  │
  └── Posts section
        ├── "POSTS"  [Grid view icon]  [List view icon]
        └── Post list (list view) or media grid (grid view)
```

---

## Components

### 1. Pet Info Card

| Field | Display |
|-------|---------|
| `avatarUrl` | Pet avatar (circular) |
| `name` | Pet name (bold) |
| `breed` | Breed name; show species if breed is null |
| `species` | Species name (muted, shown below breed) |

No action buttons — Pet Posts is a read-only view.

---

### 2. Posts List

- Same post card format as Explore — all canonical tap interactions apply (see `screen_1_home_explore.md` → Post Card):
  - Tap **family name** → Family Posts screen
  - Tap **author name** → User Posts screen
  - Tap **pet badge** on a media frame → Pet Posts screen for that pet (may be same screen if same pet)
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

**Empty state:** Possible if all posts are private and viewer is not a member — show "No posts yet".

---

## API Endpoints Required

Reuses endpoint defined in `screen_3_family_posts.md`.

### T. Query: `PetPosts`

See full definition in `screen_3_family_posts.md` → **Section: T. Query: PetPosts**.

**Variables:** `{ "petId": "<petId>", "limit": 10 }`

---

## User Flow Diagrams

### Open Pet Posts screen

```
User taps pet badge on a post card media
  └─> PetPosts query (T) { petId, first: 10 }
        ├─ PET_NOT_FOUND → show "Pet not found" error state
        └─ 200 → render pet info card + posts list
```

### Infinite Scroll

```
User scrolls to bottom
  └─> PetPosts query (T) { petId, after: <endCursor>, first: 10 }
        └─> Append new posts
              └─> hasNextPage=false → show "No more posts" state
```

### Switch to Grid View

```
User taps grid view icon
  └─> Re-render existing loaded posts as 3-column grid (no new API call)
        └─> Scroll to load more → PetPosts query (T) { petId, after: <endCursor> }
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| All posts are private, viewer is not a member | Empty state: "No posts yet" |
| Pet has no posts at all | Empty state: "No posts yet" |
| Pet is private (`isPublic = false`) | Server returns `type=random` in mediaTag to non-members; pet badge is shown as random — Pet Posts screen is never navigated to from that badge for non-members |
| `breed` is null | Show species name in the breed field; omit species sub-label |
| Pet deleted (soft delete) | Pet still exists in DB; `PetPosts` query still returns data; pet info card still renders using stored data |
| Tap same pet badge within this screen | No navigation — already on this pet's screen |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Show post count in header | Not shown |
| 2 | Show family info in header | Not shown — family name visible on each post card |
| 3 | Default post view | List view |
| 4 | Private pet visibility to non-members | mediaTag returns `type=random` → badge is not tappable to Pet Posts; non-members cannot reach this screen for private pets |
