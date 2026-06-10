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
| `created_at` | Time string following the same display rules as post cards (see `screen_1_home_explore.md` → Post Card → Time display rules). E.g. `"5m"`, `"3h"`, `"2d"`, `"28 May"` |
| `reply_count` | Total number of replies to this comment |
| `is_own` | Boolean — whether the current viewer authored this comment |
| `is_deletable` | Boolean — server-computed; `true` only when `is_own=true` AND `reply_count=0` AND comment was created within the last 10 minutes |

**Comment actions:**
- **Reply** button (always visible) → sets reply context in fixed input bar (requires login to submit)
- **Delete** button → visible only when `is_deletable = true`; tapping shows confirmation before `DeleteComment mutation (P)`

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
- Tap Send → `CreateComment mutation (L)` (top-level) or `CreateReply mutation (N)` (reply)
- Optimistic update: new comment/reply appears immediately in the list
- On API error: remove optimistic item, restore input text, show error toast
- After successful submit: clear input, scroll to the new comment/reply

---

## API Endpoints Required

> All calls go to `POST /graphql`.  
> Shared mutations from Screen 1: `LovePost`, `UnlovePost`, `HidePost`, `UpdatePost`, `DeletePost`, `PostComments`, `CreateComment`.  
> New operations specific to this screen are listed below.

---

### M. Query: `Post`

Fetch a single post for the detail view.

**Auth:** Optional — when a valid Bearer token is provided, `isLoved` and `isOwn` fields are populated on the returned post.

**Operation:**
```graphql
query Post($id: ID!) {
  post(id: $id) {
    id
    family {
      id
      name
      avatarUrl
      type
    }
    author {
      id
      displayName
      avatarUrl
    }
    pets {
      id
      name
      avatarUrl
    }
    caption
    location {
      city
      cityCode
      country
      countryCode
    }
    media {
      id
      type
      url
      thumbnailUrl
      mimeType
      width
      height
      durationSeconds
      provider
      mediaTag {
        type
        id
        species
        breed
      }
    }
    loveCount
    commentCount
    isLoved
    isOwn
    privacy
    createdAt
  }
}
```

**Variables:**
```json
{ "id": "post_001" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "post": {
      "id": "post_001",
      "family": {
        "id": "fam_xyz",
        "name": "Minh's Family",
        "avatarUrl": "https://cdn.petapp.com/families/fam_xyz/avatar.jpg",
        "type": "STANDARD"
      },
      "author": {
        "id": "user_001",
        "displayName": "Minh Tuan",
        "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
      },
      "pets": [
        { "id": "pet_111", "name": "Bụi", "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg" }
      ],
      "caption": "Bụi nằm chờ mama nấu cơm 🌕",
      "location": { "city": "Hồ Chí Minh", "cityCode": "HCM", "country": "Việt Nam", "countryCode": "VN" },
      "media": [
        {
          "id": "media_001",
          "type": "UPLOADED",
          "url": "https://cdn.petapp.com/media/001.jpg",
          "thumbnailUrl": null,
          "mimeType": "image/jpeg",
          "width": 1080,
          "height": 1080,
          "durationSeconds": null,
          "provider": null,
          "mediaTag": { "type": "PET", "id": "pet_111", "species": "Cat", "breed": "Orange Tabby Cat" }
        }
      ],
      "loveCount": 12,
      "commentCount": 4,
      "isLoved": false,
      "isOwn": false,
      "privacy": "PUBLIC",
      "createdAt": "2026-06-06T08:00:00Z"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `POST_NOT_FOUND` | Post does not exist or has been deleted |
| `200` | `FORBIDDEN` | Post exists but viewer does not have permission (e.g. `privacy=PRIVATE`, not a family member) |

---

### N. Mutation: `CreateReply`

Reply to a comment (or to a reply — any depth).

> **Triggers notification:** fires a `NEW_REPLY` notification to the author of the comment/reply being replied to (see screen_22 — Notifications screen). Not fired when replying to your own comment.

**Auth:** Required → `UNAUTHENTICATED` error if not logged in

**Operation:**
```graphql
mutation CreateReply($input: CreateReplyInput!) {
  createReply(input: $input) {
    id
    parentId
    author {
      id
      displayName
      avatarUrl
    }
    body
    createdAt
    replyCount
    isOwn
    isDeletable
  }
}
```

**Variables:**
```json
{
  "input": {
    "commentId": "comment_001",
    "body": "Đồng ý nè 😄"
  }
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "createReply": {
      "id": "reply_001",
      "parentId": "comment_001",
      "author": {
        "id": "user_003",
        "displayName": "Luna",
        "avatarUrl": "https://cdn.petapp.com/users/user_003/avatar.jpg"
      },
      "body": "Đồng ý nè 😄",
      "createdAt": "2026-06-06T09:00:00Z",
      "replyCount": 0,
      "isOwn": true,
      "isDeletable": true
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `UNAUTHENTICATED` | Caller is not logged in |
| `200` | `COMMENT_NOT_FOUND` | Target comment does not exist |

---

### O. Query: `CommentReplies`

Fetch replies for a comment (triggered when user expands "View N replies").

**Auth:** Optional

**Operation:**
```graphql
query CommentReplies($commentId: ID!, $limit: Int, $cursor: String) {
  commentReplies(commentId: $commentId, limit: $limit, cursor: $cursor) {
    replies {
      id
      parentId
      author {
        id
        displayName
        avatarUrl
      }
      body
      createdAt
      replyCount
      isOwn
      isDeletable
    }
    totalCount
    nextCursor
    hasMore
  }
}
```

**Variables:**
```json
{
  "commentId": "comment_001",
  "limit": 5,
  "cursor": null
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "commentReplies": {
      "replies": [
        {
          "id": "reply_001",
          "parentId": "comment_001",
          "author": {
            "id": "user_003",
            "displayName": "Luna",
            "avatarUrl": "https://cdn.petapp.com/users/user_003/avatar.jpg"
          },
          "body": "Đồng ý nè 😄",
          "createdAt": "2026-06-06T09:00:00Z",
          "replyCount": 2,
          "isOwn": false,
          "isDeletable": false
        }
      ],
      "totalCount": 8,
      "nextCursor": "eyJpZCI6InJlcGx5XzAwMSJ9",
      "hasMore": true
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `COMMENT_NOT_FOUND` | Parent comment does not exist |

---

### P. Mutation: `DeleteComment`

Delete a comment or reply.

**Auth:** Required → `UNAUTHENTICATED` error if not logged in

**Server validates all three conditions before deleting:**
1. Caller must be the comment author (`isOwn`)
2. Comment must have no replies (`replyCount = 0`)
3. Comment must have been created within the last **10 minutes**

If any condition fails → `FORBIDDEN` error.

**Behaviour:** Hard delete only — comment is removed from the list entirely. No soft delete / "[deleted]" placeholder.

**Operation:**
```graphql
mutation DeleteComment($input: DeleteCommentInput!) {
  deleteComment(input: $input) {
    success
  }
}
```

**Variables:**
```json
{ "input": { "commentId": "comment_001" } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "deleteComment": {
      "success": true
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `UNAUTHENTICATED` | Caller is not logged in |
| `200` | `NOT_COMMENT_AUTHOR` | Caller did not author this comment |
| `200` | `COMMENT_HAS_REPLIES` | Comment already has one or more replies |
| `200` | `DELETE_WINDOW_EXPIRED` | More than 10 minutes have passed since comment was created |
| `200` | `COMMENT_NOT_FOUND` | Comment does not exist |

---

## User Flow Diagrams

### Open Post Detail

```
User taps post in Explore feed
  └─> Post query (M) { id: post_id }
        ├─ POST_NOT_FOUND → show "Post not found" error screen
        ├─ FORBIDDEN → show "You don't have permission to view this post"
        └─ 200 → render post card
              └─> PostComments query (K) { postId, limit: 20 }
                    └─> Render top-level comments
                          └─> Each comment with replyCount > 0 shows "View N replies ▾"
```

### Expand Replies

```
User taps "View N replies ▾"
  └─> CommentReplies query (O) { commentId, limit: 5 }
        └─> Render first 5 replies under comment
              ├─ hasMore=true → show "Load N more replies"
              └─ Each reply with replyCount > 0 → shows "View N replies ▾" (recursive)
```

### Submit Comment

```
User types in input bar → taps Send
  └─> [unauthenticated] → redirect to Login
  └─> [authenticated, top-level] → CreateComment mutation (L) { postId, body }
  └─> [authenticated, replying]  → CreateReply mutation (N) { parentId, body }
        ├─ Optimistic: prepend new item immediately
        ├─ Success: confirm item, clear input
        └─ Error: remove item, restore input text, show toast
```

### Delete Comment

```
User taps Delete on own comment  (button only visible when isDeletable=true)
  └─> Show confirmation dialog
        └─> Confirm → DeleteComment mutation (P) { commentId }
              ├─ Success → remove comment from list; decrement parent replyCount if reply
              └─ COMMENT_HAS_REPLIES / DELETE_WINDOW_EXPIRED
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
| Deep link to post | Load `Post query (M)` directly; Back button returns to previous screen or Explore if no history |
| Tap family name on post | Navigate to Family Posts screen |
| Tap author name on post | Navigate to User Posts screen |
| Tap pet badge on media | Navigate to Pet Posts screen |
