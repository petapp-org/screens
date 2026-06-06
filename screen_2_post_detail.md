# Screen 2: Post Detail

## Overview

Full-screen view of a single post with complete comment thread.  
Accessible without login — unauthenticated users can read the post and all comments.  
Navigated to from: Explore feed (tap post body/media), inline comment panel ("View all N comments" link), pet badge, or direct deep link.

---

## UI Layout

```
[Header]
  Left: Back button
  Center: "Post" (static title)
  Right: ... (context menu — same rules as Explore)

[Scrollable area]
  ├── Post Card (identical to Explore post card)
  │     ├── Family avatar + family name + author name + pets subtitle + timestamp
  │     ├── Media carousel (uploaded / embedded, pet badge per frame)
  │     ├── Caption
  │     └── Love count · Comment count  [Love button]
  │
  └── Comments section
        ├── Comment 1
        │     ├── [Reply button]  [Delete button — own comment only]
        │     └── Replies (nested, collapsible — see below)
        ├── Comment 2
        │     └── ...
        └── [Load more comments]

[Fixed bottom bar — always visible]
  [User avatar]  [Text input: "Add a comment..."]  [Send button]
  └── If replying: shows "Replying to @username ×" above input
```

---

## Components

### 1. Post Card

Identical to the canonical post card defined in `screen_1_home_explore.md` → **Section: Post Card**. All fields, layout rules, and interactions apply including:
- Header: family avatar · family name (top-left) | `author_name · time` (top-right) | pets subtitle (bottom-left) | location (bottom-right, if set)
- Media carousel with `N/Total` badge and per-frame pet badge (`media_tag.type = "pet"` only)
- Tap **pet badge** → Pet Posts screen
- Tap **family name** → Family Posts screen
- Tap **author name** → User Posts screen
- Love button with optimistic update
- `...` context menu (same 3-case logic as Explore)

---

### 2. Comments Section

**Top-level comments:**
- Sorted by `created_at` asc (oldest first)
- Paginated: 20 per page, "Load more" button at bottom of list
- Each comment shows:

| Field | Description |
|-------|-------------|
| `author.avatar_url` | Commenter's avatar |
| `author.display_name` | Commenter's name — tappable → User Profile screen |
| `body` | Comment text |
| `created_at` | Relative time (e.g. "2h ago") |
| `reply_count` | Total number of replies to this comment |
| `is_own` | Boolean — whether the current viewer authored this comment |
| `is_deletable` | Boolean — server-computed; `true` only when `is_own=true` AND `reply_count=0` AND comment was created within the last 10 minutes |

**Comment actions:**
- **Reply** button (always visible) → sets reply context in fixed input bar (requires login to submit)
- **Delete** button → visible only when `is_deletable = true`; tapping shows confirmation before `DELETE /comments/{comment_id}`

**Commenting requires login:**
- Unauthenticated users can read all comments and replies
- Tapping the input bar, Reply button, or Send → redirect to Login

---

### 3. Nested Replies

Replies are nested under their parent comment. Multi-level nesting is supported (reply to a reply, and so on).

**Initial state:**
- If `reply_count > 0`: show a collapsed "View N replies ▾" link under the comment
- Tapping expands and loads the first 5 replies

**Loaded reply:**
- Same display fields as a top-level comment (`author`, `body`, `created_at`, `is_own`, `is_deletable`)
- Has its own **Reply** and **Delete** buttons (same rules — `is_deletable` applies identically)
- If a reply itself has replies, show "View N replies ▾" beneath it (same expand behaviour, recursively)

**"Load more replies":**
- If a comment has > 5 replies, show "Load N more replies" after the first 5

**Collapsing:**
- Tapping "View N replies ▾" again collapses the reply thread

---

### 4. Fixed Comment Input Bar

Always pinned to the bottom of the screen, above the system navigation bar.

**States:**

| State | Display |
|-------|---------|
| Default | Placeholder: "Add a comment..." |
| Replying to a comment | Banner above input: "Replying to @username ×" — tap × to cancel reply |
| Unauthenticated | Input and Reply buttons are tappable → redirect to Login; Send button disabled |
| Authenticated, empty input | Send button disabled |
| Authenticated, non-empty input | Send button enabled |

**Submit behaviour:**
- Tap Send → `POST /posts/{post_id}/comments` (top-level) or `POST /comments/{parent_id}/replies` (reply)
- Optimistic update: new comment/reply appears immediately in the list
- On API error: remove optimistic item, restore input text, show error toast
- After successful submit: clear input, scroll to the new comment/reply

---

## API Endpoints Required

> Endpoints F, G (love/unlove), H (hide), I (edit post), J (delete post) are shared with Screen 1.  
> Endpoints K, L (list/create comments) are shared with Screen 1.  
> New endpoints specific to this screen are listed below.

---

### M. `GET /posts/{post_id}`

Fetch a single post for the detail view.

**Headers:**
- `Authorization: Bearer <token>` — optional. Populates `is_loved`, `is_own` on comments.

**Response `200 OK`:** Same post object shape as `GET /feed/explore` response item.

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| `404` | `POST_NOT_FOUND` | Post does not exist or has been deleted |
| `403` | `FORBIDDEN` | Post exists but viewer does not have permission (e.g. `privacy=private`, not a family member) |

---

### N. `POST /comments/{comment_id}/replies`

Reply to a comment (or to a reply — any depth).

**Auth:** Required → `401` if not logged in

**Path param:** `comment_id` — the comment being replied to (can be a top-level comment or an existing reply)

**Body:**
```json
{ "body": "Đồng ý nè 😄" }
```

**Response `201 Created`:**
```json
{
  "id": "reply_001",
  "parent_id": "comment_001",
  "author": {
    "id": "user_003",
    "display_name": "Luna",
    "avatar_url": "https://cdn.petapp.com/users/user_003/avatar.jpg"
  },
  "body": "Đồng ý nè 😄",
  "created_at": "2026-06-06T09:00:00Z",
  "reply_count": 0,
  "is_own": true
}
```

---

### O. `GET /comments/{comment_id}/replies`

Fetch replies for a comment (triggered when user expands "View N replies").

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `limit` | int | No | `5` | Replies per page |
| `cursor` | string | No | — | Pagination cursor for "Load more replies" |

**Response `200 OK`:**
```json
{
  "replies": [
    {
      "id": "reply_001",
      "parent_id": "comment_001",
      "author": {
        "id": "user_003",
        "display_name": "Luna",
        "avatar_url": "https://cdn.petapp.com/users/user_003/avatar.jpg"
      },
      "body": "Đồng ý nè 😄",
      "created_at": "2026-06-06T09:00:00Z",
      "reply_count": 2,
      "is_own": false
    }
  ],
  "total_count": 8,
  "next_cursor": "eyJpZCI6InJlcGx5XzAwMSJ9",
  "has_more": true
}
```

---

### P. `DELETE /comments/{comment_id}`

Delete a comment or reply.

**Auth:** Required → `401` if not logged in

**Server validates all three conditions before deleting:**
1. Caller must be the comment author (`is_own`)
2. Comment must have no replies (`reply_count = 0`)
3. Comment must have been created within the last **10 minutes**

If any condition fails → `403`.

**Behaviour:** Hard delete only — comment is removed from the list entirely. No soft delete / "[deleted]" placeholder.

**Response `204 No Content`**

**Error Responses:**

| Status | Code | Scenario |
|--------|------|----------|
| `403` | `NOT_COMMENT_AUTHOR` | Caller did not author this comment |
| `403` | `COMMENT_HAS_REPLIES` | Comment already has one or more replies |
| `403` | `DELETE_WINDOW_EXPIRED` | More than 10 minutes have passed since comment was created |
| `404` | `COMMENT_NOT_FOUND` | Comment does not exist |

---

## User Flow Diagrams

### Open Post Detail

```
User taps post in Explore feed
  └─> GET /posts/{post_id}
        ├─ 404 → show "Post not found" error screen
        ├─ 403 → show "You don't have permission to view this post"
        └─ 200 → render post card
              └─> GET /posts/{post_id}/comments?limit=20
                    └─> Render top-level comments
                          └─> Each comment with reply_count > 0 shows "View N replies ▾"
```

### Expand Replies

```
User taps "View N replies ▾"
  └─> GET /comments/{comment_id}/replies?limit=5
        └─> Render first 5 replies under comment
              ├─ has_more=true → show "Load N more replies"
              └─ Each reply with reply_count > 0 → shows "View N replies ▾" (recursive)
```

### Submit Comment

```
User types in input bar → taps Send
  └─> [unauthenticated] → redirect to Login
  └─> [authenticated, top-level] → POST /posts/{post_id}/comments
  └─> [authenticated, replying]  → POST /comments/{parent_id}/replies
        ├─ Optimistic: prepend new item immediately
        ├─ Success: confirm item, clear input
        └─ Error: remove item, restore input text, show toast
```

### Delete Comment

```
User taps Delete on own comment  (button only visible when is_deletable=true)
  └─> Show confirmation dialog
        └─> Confirm → DELETE /comments/{comment_id}
              ├─ 204 → remove comment from list; decrement parent reply_count if it's a reply
              └─ 403 COMMENT_HAS_REPLIES / DELETE_WINDOW_EXPIRED
                    → hide Delete button immediately; show toast "Comment can no longer be deleted"
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Unauthenticated user | Can view post and all comments; tap input / Reply → redirect to Login |
| Post not found (deleted) | Show "Post not found" error state with Back button |
| Post `privacy=private`, viewer not a family member | Show `403` error state |
| Post `privacy=followers`, viewer not following | Show `403` error state |
| Delete attempted after 10 min window or comment has replies | `403` returned; hide Delete button; show toast |
| Comment deleted successfully | Removed from list; parent `reply_count` decremented if reply |
| Input bar — replying context | Banner "Replying to @username ×" shown above input; tap × clears reply context |
| No comments yet | Show empty state: "Be the first to comment" |
| Deep link to post | Load `GET /posts/{post_id}` directly; Back button returns to previous screen or Explore if no history |
| Tap family name on post | Navigate to Family Posts screen |
| Tap author name on post | Navigate to User Posts screen |
| Tap pet badge on media | Navigate to Pet Posts screen |
