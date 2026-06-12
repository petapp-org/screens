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
        │     └── Replies (depth-2 flat list, collapsible — see below)
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
- Header: family avatar · family name (top-left) | `authorName · time` (top-right) | pets subtitle (bottom-left) | location (bottom-right, if set)
- Media carousel with `N/Total` badge and per-frame pet badge (`mediaTag.type = PET` only)
- Tap **pet badge** → Pet Posts screen

> **`focusPetId` (optional nav param):** when Post Detail is opened from a pet's **Story** (`screen_29`), it carries `focusPetId`. The media carousel is then **reordered client-side** so the media tagged to that pet (`mediaTag.type = PET`, `petId = focusPetId`) come **first** (preserving their relative order), then the rest. No API change — uses the `media[].mediaTag` already returned by `Post (M)`. Without the param, media keep their original post order.
- Tap **family name** → Family Posts screen
- Tap **author name** → User Posts screen
- Love button with optimistic update
- `...` context menu (same 3-case logic as Explore)

---

### 2. Comments Section

**Top-level comments:**
- Sorted by `createdAt` asc (oldest first)
- Paginated: 20 per page, "Load more" button at bottom of list
- Each comment shows:

| Field | Description |
|-------|-------------|
| `author.avatarUrl` | Commenter's avatar |
| `author.displayName` | Commenter's name — tappable → User Profile screen |
| `body` | Comment text |
| `createdAt` | Time string following the same display rules as post cards (see `screen_1_home_explore.md` → Post Card → Time display rules). E.g. `"5m"`, `"3h"`, `"2d"`, `"28 May"` |
| `replyCount` | Total number of replies to this comment |
| `isOwn` | Boolean — whether the current viewer authored this comment |
| `isDeletable` | Boolean — server-enforced (computed per-viewer by `app-community`, **not** the client); `true` only when `isOwn=true` AND `replyCount=0` AND comment was created within the last 10 minutes. The same rule gates the `DeleteComment` mutation, so `isDeletable=true` guarantees the delete will succeed |

> ✅ replyCount / isDeletable / replies / replyToUser: shipped theo decision depth-2 + @mention (petapp-be #880, epic #948 — sub-issues #949/#950/#951/#952 đã merge). isOwn đã có (PR #879). `isDeletable` is **server-enforced** (computed per-viewer trong `app-community`, cùng rule gating `DeleteComment`), không phải client-computed.

**Comment actions:**
- **Reply** button (always visible) → sets reply context in fixed input bar (requires login to submit)
- **Delete** button → visible only when `isDeletable = true`; tapping shows confirmation before `DeleteComment mutation (P)`

**Commenting requires login:**
- Unauthenticated users can read all comments and replies
- Tapping the input bar, Reply button, or Send → redirect to Login

---

### 3. Replies (depth-2, Facebook-style)

Threading is **2 levels deep** (top-level comment → replies), **not** infinitely recursive. This matches the Facebook model and is the backend-supported shape (petapp-be #880 / epic #948).

- A top-level comment (depth 0) can have a flat list of replies (depth 1).
- Replying to a reply does **not** create a third level — the new reply is appended to the **same** flat reply list of the top-level comment, with an automatic **`@mention`** of the user being replied to (`replyToUser`) prepended so the "who replied to whom" context is preserved.
- So every reply lives under exactly one top-level comment; there are no nested "View replies" links inside replies.

**Initial state:**
- If `replyCount > 0`: show a collapsed "View N replies ▾" link under the top-level comment
- Tapping expands and loads the first 5 replies (via `CommentReplies` query — see O)

**Loaded reply (depth 1):**
- Same display fields as a top-level comment (`author`, `body`, `createdAt`, `isOwn`, `isDeletable`)
- When the reply targets another reply, it renders a leading `@username` (`replyToUser`) — tappable to scroll to that user's reply
- Has its own **Reply** and **Delete** buttons (same rules — `isDeletable` applies identically)
- A reply does **not** show its own "View N replies" link (no depth-3)

**"Load more replies":**
- If a top-level comment has > 5 replies, show "Load N more replies" after the first 5 (cursor `after`)

**Collapsing:**
- Tapping "View N replies ▾" again collapses the reply list

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
- Tap Send → `CreateComment mutation (L)` (top-level, `parentId: null`) or `CreateComment mutation (N)` (reply, `parentId: commentId`)
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
      familyType
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
    body
    location {
      city
      cityCode
      country
      countryCode
    }
    media {
      order
      sourceType
      embedUrl
      embedProvider
      mediaTag {
        type
        petId
        species
        breed
      }
      media {
        id
        type
        thumbnailUrl
        variants { size key contentType }
        hlsUrl
        blurhash
        signedUrl
        expiresAt
      }
    }
    loveCount
    commentsCount
    isLoved
    isOwn
    visibility
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
        "familyType": "NORMAL"
      },
      "author": {
        "id": "user_001",
        "displayName": "Minh Tuan",
        "avatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg"
      },
      "pets": [
        { "id": "pet_111", "name": "Bụi", "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg" }
      ],
      "body": "Bụi nằm chờ mama nấu cơm 🌕",
      "location": { "city": "Hồ Chí Minh", "cityCode": "HCM", "country": "Việt Nam", "countryCode": "VN" },
      "media": [
        {
          "order": 1,
          "sourceType": "UPLOADED",
          "embedUrl": null,
          "embedProvider": null,
          "mediaTag": { "type": "PET", "petId": "pet_111", "species": "Cat", "breed": "Orange Tabby Cat" },
          "media": {
            "id": "media_001",
            "type": "IMAGE",
            "thumbnailUrl": null,
            "variants": [{ "size": "1080x1080", "key": "media/001_1080.jpg", "contentType": "image/jpeg" }],
            "hlsUrl": null,
            "blurhash": "L6PZfSi_.AyE_3t7t7R**0o#DgR4",
            "signedUrl": "https://cdn.petapp.com/media/001.jpg?token=abc",
            "expiresAt": "2026-06-13T08:00:00Z"
          }
        }
      ],
      "loveCount": 12,
      "commentsCount": 4,
      "isLoved": false,
      "isOwn": false,
      "visibility": "PUBLIC",
      "createdAt": "2026-06-06T08:00:00Z"
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `POST_NOT_FOUND` | Post does not exist or has been deleted |
| `200` | `FORBIDDEN` | Post exists but viewer does not have permission (e.g. `visibility=PRIVATE`, not a family member) |

---

### N. Mutation: `CreateComment` (reply form)

Reply to a top-level comment or to a reply (depth-2 model). Uses the unified `createComment` mutation. When replying to a **top-level comment**, set `parentId` to that comment's id. When replying to a **reply**, set `parentId` to the **top-level comment** that owns the thread and `replyToUserId` to the replied-to user — the backend keeps the reply at depth 2 and prepends an `@mention` (no depth-3 is created).

> **Triggers notification:** fires a `NEW_REPLY` notification to the author of the comment/reply being replied to (see screen_22 — Notifications screen). Not fired when replying to your own comment.

**Auth:** Required → `UNAUTHENTICATED` error if not logged in

**Operation:**
```graphql
mutation CreateComment($postId: ID!, $body: String!, $parentId: ID = null, $replyToUserId: ID = null) {
  createComment(postId: $postId, body: $body, parentId: $parentId, replyToUserId: $replyToUserId) {
    id
    parentId
    author {
      id
      displayName
      avatarUrl
    }
    replyToUser {
      id
      displayName
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
  "postId": "post_001",
  "body": "Đồng ý nè 😄",
  "parentId": "comment_001"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "createComment": {
      "id": "reply_001",
      "parentId": "comment_001",
      "author": {
        "id": "user_003",
        "displayName": "Luna",
        "avatarUrl": "https://cdn.petapp.com/users/user_003/avatar.jpg"
      },
      "replyToUser": {
        "id": "user_001",
        "displayName": "Minh Tuan"
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

> **Shape note:** there is no root `commentReplies` query in the contract. Replies are accessed via the sub-field `Comment.replies(first, after): CommentConnection!` (shipped). The total reply count is `Comment.replyCount` (already fetched as part of `PostComments (K)`). The operation below fetches the parent post's comments and expands replies for the target comment in one query, or a client may re-fetch the specific comment thread after load; either approach uses the same sub-field path.

**Auth:** Optional

**Operation:**
```graphql
query CommentReplies($postId: ID!, $commentId: ID!, $first: Int! = 5, $after: String) {
  post(id: $postId) {
    comments(first: 1, after: null) {
      edges {
        node {
          id
          replyCount
          replies(first: $first, after: $after) {
            edges {
              cursor
              node {
                id
                parentId
                author {
                  id
                  displayName
                  avatarUrl
                }
                replyToUser {
                  id
                  displayName
                }
                body
                createdAt
                replyCount
                isOwn
                isDeletable
              }
            }
            pageInfo {
              hasNextPage
              endCursor
            }
          }
        }
      }
    }
  }
}
```

> **Implementation note:** in practice the client already has the parent comment node in cache (from `PostComments K`). A focused fetch can target the specific comment's `replies(first, after)` sub-field directly. The `replyCount` used for the "View N replies" label comes from the parent `Comment.replyCount` already in the list. No root `commentReplies` field is needed.

**Variables:**
```json
{
  "postId": "post_001",
  "commentId": "comment_001",
  "first": 5,
  "after": null
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "post": {
      "comments": {
        "edges": [
          {
            "node": {
              "id": "comment_001",
              "replyCount": 8,
              "replies": {
                "edges": [
                  {
                    "cursor": "eyJpZCI6InJlcGx5XzAwMSJ9",
                    "node": {
                      "id": "reply_001",
                      "parentId": "comment_001",
                      "author": {
                        "id": "user_003",
                        "displayName": "Luna",
                        "avatarUrl": "https://cdn.petapp.com/users/user_003/avatar.jpg"
                      },
                      "replyToUser": null,
                      "body": "Đồng ý nè 😄",
                      "createdAt": "2026-06-06T09:00:00Z",
                      "replyCount": 0,
                      "isOwn": false,
                      "isDeletable": false
                    }
                  }
                ],
                "pageInfo": {
                  "hasNextPage": true,
                  "endCursor": "eyJpZCI6InJlcGx5XzAwMSJ9"
                }
              }
            }
          }
        ]
      }
    }
  }
}
```

> **Note:** total reply count for the parent comment is `Comment.replyCount` — a field on the comment node, not inside the connection. This follows ADR-0023 (no `totalCount` inside the Relay envelope).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `200` | `POST_NOT_FOUND` | Post does not exist |
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
mutation DeleteComment($commentId: ID!) {
  deleteComment(commentId: $commentId) {
    commentId
    deleted
  }
}
```

**Variables:**
```json
{ "commentId": "comment_001" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "deleteComment": {
      "commentId": "comment_001",
      "deleted": true
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
  └─> Post query (M) { id: postId }
        ├─ POST_NOT_FOUND → show "Post not found" error screen
        ├─ FORBIDDEN → show "You don't have permission to view this post"
        └─ 200 → render post card
              └─> PostComments query (K) { postId, first: 20 }
                    └─> Render top-level comments
                          └─> Each comment with replyCount > 0 shows "View N replies ▾"
```

### Expand Replies

```
User taps "View N replies ▾"
  └─> CommentReplies query (O) — fetches Comment.replies(first: 5) sub-field { commentId, first: 5 }
        └─> Render first 5 replies under the top-level comment (flat, depth-2)
              ├─ hasNextPage=true → show "Load N more replies"
              └─ Replies have NO own "View replies" link (no depth-3); cross-reply context shown via @mention
```

### Submit Comment

```
User types in input bar → taps Send
  └─> [unauthenticated] → redirect to Login
  └─> [authenticated, top-level] → CreateComment mutation (L) { postId, body }
  └─> [authenticated, replying]  → CreateComment mutation (N) { postId, body, parentId }
        ├─ Optimistic: prepend new item immediately
        ├─ Success: confirm item, clear input
        └─ Error: remove item, restore input text, show toast
```

### Delete Comment

```
User taps Delete on own comment  (button only visible when isDeletable = true)
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
| Post `visibility=private`, viewer not a family member | Show `403` error state |
| Post `visibility=followers`, viewer not following | Show `403` error state |
| Delete attempted after 10 min window or comment has replies | `403` returned; hide Delete button; show toast |
| Comment deleted successfully | Removed from list; parent `replyCount` decremented if reply |
| Input bar — replying context | Banner "Replying to @username ×" shown above input; tap × clears reply context |
| No comments yet | Show empty state: "Be the first to comment" |
| Deep link to post | Load `Post query (M)` directly; Back button returns to previous screen or Explore if no history |
| Tap family name on post | Navigate to Family Posts screen |
| Tap author name on post | Navigate to User Posts screen |
| Tap pet badge on media | Navigate to Pet Posts screen |
