# Screen 15: Loved Posts

## Overview

Private feed of posts the current user has loved, sorted by most recently loved first.  
Navigated to from: **Profile Settings screen (screen_5)** → Activity → Loves.  
Requires login — unauthenticated users cannot reach this screen (Profile Settings redirects to Login).  
Private — only the current user can view their own loved posts.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Loved Posts" (static)

[Scrollable area]
  └── Post list
        └── [Canonical post cards — infinite scroll]

[Empty state — shown when no loved posts]
  [Heart icon]
  "You haven't loved any posts yet"
```

---

## Components

### 1. Posts List

- Same post card format as Explore — all canonical tap interactions apply (see `screen_1_home_explore.md` → Post Card):
  - Tap **family name** → Family Posts screen
  - Tap **author name** → User Posts screen
  - Tap **pet badge** → Pet Posts screen
  - Tap **post body / media** → Post Detail screen
- **Love button behavior** on this screen: tapping Love → **un-love**
  - Optimistic: remove post row immediately
  - On API error: revert (re-insert post at same position), show error toast
- Sorted by love date desc (most recently loved first) — not by post creation date
- 10 posts per page, infinite scroll
- No filter tabs; no Suggested Families widget
- No list/grid toggle — list view only

---

## API Endpoints Required

Reuses endpoint defined in `screen_5_profile_settings.md`.

### AH. Query: `MyLovedPosts`

See full definition in `screen_5_profile_settings.md` → **Section: AH. Query: MyLovedPosts**.

**Variables:** `{ "first": 20, "after": null }`

Unlove uses existing endpoints from `screen_1_home_explore.md`:

### G. Mutation: `UnlovePost`

See full definition in `screen_1_home_explore.md` → **Section: G. Mutation: UnlovePost**.

---

## User Flow Diagrams

### Open Loved Posts screen

```
User taps Activity → Loves in Profile Settings
  └─> MyLovedPosts query (AH) { first: 20, after: null }
        └─> 200 → render post list
              └─> list empty → show empty state
```

### Infinite Scroll

```
User scrolls to bottom
  └─> MyLovedPosts query (AH) { first: 20, after: pageInfo.endCursor }
        └─> Append new posts
              └─> pageInfo.hasNextPage=false → show "No more posts" state
```

### Un-love a Post

```
User taps Love button on a post in this list
  └─> Optimistic: remove post from list
        └─> UnlovePost mutation (G) { post { isLoved loveCount } }
              ├─ Success → post removed confirmed
              └─ Error → re-insert post, show error toast
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| No loved posts | Empty state: "You haven't loved any posts yet" |
| Post was deleted by owner after user loved it | Server returns it filtered out; gap in list is silent |
| Un-love: API error after optimistic remove | Post re-inserted at original position; error toast shown |
| Post's privacy changed to private after user loved it | Server may exclude it from results — gap is silent |
| Tap Love button again after undo | Post is re-loved (LovePost mutation F); post re-appears on next list refresh |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Grid view toggle | Not shown — list view only |
| 2 | Filter tabs | Not shown |
| 3 | Suggested Families widget | Not shown |
| 4 | Sort order | Love date desc (most recently loved first), not post creation date |
| 5 | Accessibility | Private — only current user; no public equivalent |
