# Screen 16: Following

## Overview

List of all families the current user follows, sorted by most recently followed first.  
Navigated to from: **Profile Settings screen (screen_5)** → Activity → Following.  
Requires login — unauthenticated users cannot reach this screen (Profile Settings redirects to Login).

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Following" (static)

[Scrollable area]
  └── Family rows list
        ├── Family row 1
        ├── Family row 2
        └── ...
        [Load more button — appears when pageInfo.hasNextPage=true]

[Empty state — shown when following no families]
  [Icon]
  "You're not following anyone yet"
  [Suggested Families widget]
```

---

## Components

### 1. Family Row

Each row contains:

| Element | Description |
|---------|-------------|
| Family avatar (circular) | Left side |
| Family name | Bold, primary text |
| `@tag` | Below family name, muted |
| City, Country | Below tag, muted |
| **Following button** | Right side — always shown |
| **Donate button** | Right side — shown only when `familyType = CHARITY` |

**Row tap (anywhere except buttons):** → navigate to **Family Posts screen** for that family.

**Following button:**
- Label: "Following"
- Tap → unfollow
  - Optimistic: remove row immediately
  - Show toast: *"Unfollowed [Family name]"* + **[Undo]** button (5s window)
  - Tap Undo → `FollowFamily mutation (D)` → re-add row at same position; toast dismissed
  - No Undo tap (or toast dismissed) → `UnfollowFamily mutation (E)` confirmed

**Donate button** (`familyType = CHARITY` only):
- Tap → **opens a chat** with that charity family (pre-filled VN support message) — interim, no wallet; canonical in `screen_3`. Login required; hidden for members of that charity.

---

### 2. Load More Button

- Pagination is **manual** — no infinite scroll; user taps "Load more" explicitly
- 20 families per page
- "Load more" button appears at bottom of list when `pageInfo.hasNextPage = true`
- Each tap loads 20 more and appends to list

---

### 3. Empty State

Shown when user follows no families (or all have been unfollowed):
- Message: "You're not following anyone yet"
- **Suggested Families widget** shown below — same component as Explore feed (see `screen_1_home_explore.md` → Suggested Families Widget); uses `SuggestedFamilies query (B)`

---

## API Endpoints Required

Reuses endpoints defined in `screen_5_profile_settings.md` and `screen_1_home_explore.md`.

### AI. Query: `MyFollowing`

See full definition in `screen_5_profile_settings.md` → **Section: AI. Query: MyFollowing**.

> ✅ **Resolved (petapp-be#921):** backend exposes `myFollowingFamilies(first, after): FamilyConnection!` (Relay). This screen renders family rows directly from it.

**Variables:** `{ "first": 20, "after": null }`

### D. Mutation: `FollowFamily` *(for Undo)*

See full definition in `screen_1_home_explore.md` → **Section: D. Mutation: FollowFamily**.

### E. Mutation: `UnfollowFamily`

See full definition in `screen_1_home_explore.md` → **Section: E. Mutation: UnfollowFamily**.

### B. Query: `SuggestedFamilies` *(empty state only)*

See full definition in `screen_1_home_explore.md` → **Section: B. Query: SuggestedFamilies**.

---

## User Flow Diagrams

### Open Following screen

```
User taps Activity → Following in Profile Settings
  └─> MyFollowing query (AI) { first: 20 }
        └─> 200 → render family rows
              └─> list empty → show empty state + Suggested Families widget
```

### Load More

```
User taps "Load more" button
  └─> MyFollowing query (AI) { after: <endCursor>, first: 20 }
        └─> Append new rows
              └─> pageInfo.hasNextPage=false → hide "Load more" button
```

### Unfollow with Undo

```
User taps Following button on a row
  └─> Optimistic: remove row
        └─> Show toast "Unfollowed [Name]" + Undo button (5s)
              ├─ Tap Undo
              │     └─> FollowFamily mutation (D) { familyId }
              │           └─> Success → re-insert row; dismiss toast
              └─> No Undo (timeout or dismiss)
                    └─> UnfollowFamily mutation (E) { family { social { isFollowedByMe } } } confirmed
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No followed families | Empty state: "You're not following anyone yet" + Suggested Families widget |
| All families unfollowed during session | Empty state appears inline after last row removed |
| Undo after toast dismissed early | Not possible — Undo button only active while toast is visible (5s window) |
| Unfollow API error | Re-insert row at original position; show error toast |
| `familyType = CHARITY` family | Show Donate button alongside Following button |
| Family deleted after user followed it | Server returns it filtered out; gap in list is silent |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Pagination type | Manual "Load more" button (not infinite scroll) — 20 per tap |
| 2 | Search/filter | Not shown |
| 3 | Unfollow confirmation | No dialog — only toast with 5s Undo (consistent with screen_3 and screen_11) |
| 4 | Empty state | Suggested Families widget shown (same as Explore) |
