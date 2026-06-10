# Screen 14: Random Pet Posts

## Overview

Post feed for all AI-detected but unmatched animals within a specific family — shows posts where at least one media frame has `mediaTag.type = "RANDOM"` with a detected breed or species.  
Navigated to from: tapping **Random Pets row** in Family Posts (only when `randomCount > 0`); tapping **"View All →"** in the Random Pets section in My Pets (screen_8, only when random posts exist).  
Accessible without login — unauthenticated users see only `public` posts.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Random Pets" (static)

[Scrollable area]
  ├── Family Info Card (minimal)
  │     ├── [Family avatar]
  │     ├── Family name
  │     └── N randoms  (e.g. "10 randoms")
  │
  └── Posts section
        ├── "POSTS"  [Grid view icon]  [List view icon]
        └── Post list (list view) or media grid (grid view)
```

---

## Components

### 1. Family Info Card (minimal)

| Field | Display |
|-------|---------|
| `avatarUrl` | Family avatar (circular) |
| `name` | Family name (bold) |
| `randomCount` | e.g. `"10 randoms"` (muted text) |

No follow/message buttons — this is a filtered view of a family's posts, not the full family profile. Tap family name → Family Posts screen for full profile.

---

### 2. Posts List

- Posts filtered to: at least one media frame has `mediaTag.type = "RANDOM" AND (breed IS NOT NULL OR species IS NOT NULL)` — AI detected an animal but did not match a named pet in the family
- Same post card format as Explore — all canonical tap interactions apply (see `screen_1_home_explore.md` → Post Card):
  - Tap **family name** → Family Posts screen
  - Tap **author name** → User Posts screen
  - Tap **post body / media** → Post Detail screen
  - **Pet badge**: `mediaTag.type = "RANDOM"` → no tappable pet badge shown on any media frame; no navigation to Pet Posts from within this screen
- Sorted `createdAt` desc (newest first)
- 10 posts per page, infinite scroll
- List/grid view toggle (same as Family Posts screen)
- Grid view: 3-column thumbnail grid; tap cell → Post Detail screen

**Privacy rules:**
- Same server-side enforcement as Explore — unauthenticated sees `public` only
- Server enforces — client receives only visible posts

**Empty state:** Not applicable — both entry points are guarded by `randomCount > 0` / section only shown when posts exist.

---

## API Endpoints Required

Reuses endpoint defined in `screen_3_family_posts.md`.

### U. Query: `RandomPetPosts`

See full definition in `screen_3_family_posts.md` → **Section: U. Query: RandomPetPosts**.

**Variables:** `{ "familyId": "<familyId>", "limit": 10 }`

---

## User Flow Diagrams

### Open Random Pet Posts screen (from Family Posts)

```
User taps Random Pets row in Family Posts
  └─> RandomPetPosts query (U) { familyId, limit: 10 }
        ├─ FAMILY_NOT_FOUND → show "Family not found" error state
        └─ 200 → render family info card + posts list
```

### Open Random Pet Posts screen (from My Pets)

```
User taps "View All →" in Random Pets section (My Pets screen)
  └─> RandomPetPosts query (U) { familyId: activeFamily.id, limit: 10 }
        └─> 200 → render family info card + posts list
```

### Infinite Scroll

```
User scrolls to bottom
  └─> RandomPetPosts query (U) { familyId, cursor: <nextCursor>, limit: 10 }
        └─> Append new posts
              └─> hasMore=false → show "No more posts" state
```

### Switch to Grid View

```
User taps grid view icon
  └─> Re-render existing loaded posts as 3-column grid (no new API call)
        └─> Scroll to load more → RandomPetPosts query (U) { familyId, cursor: <nextCursor> }
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Entry point guards | Both entry points check `randomCount > 0` before allowing navigation — empty state should never occur |
| Posts with mixed tags | A post may have some `type=PET` frames alongside `type=RANDOM` frames; it appears in this list if at least one frame is `RANDOM` with breed/species |
| `mediaTag.type = "RANDOM"` on media frames | No pet badge shown; no navigation to Pet Posts from these frames |
| Family changes between navigation events | If `randomCount` drops to 0 after navigation (race condition), list simply shows empty — acceptable edge case |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Screen title | "Random Pets" (fixed) |
| 2 | Pet badge on media | Not shown for `type=RANDOM` frames — no link to Pet Posts screen |
| 3 | Scope | Per-family only — no cross-family random posts view |
| 4 | Empty state | Not needed — entry points are guarded |
