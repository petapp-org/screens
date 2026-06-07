# Screen 10: Messages

## Overview

Inbox for all conversations — family threads and direct messages.  
Auth required. Accessible from: Messages icon in Explore header (screen_1) and My Pets header (screen_8).  
Red dot badge on Messages icon when there are any unread messages.

---

## UI Layout

```
[Header]
  Title: "Messages"
  Right: [Compose icon]  ← future; not in scope yet

[Search bar]
  Placeholder: "Search by name or @tag"
  └── Filters conversation list in real-time by user/family name or @tag
  └── Min 2 characters to trigger; debounce 300ms
  └── Matches across all sections (My Families, Sent to Families, DM)
  └── No results → show "No conversations found"

━━ Family Conversations ━━━━━━━━━━━━━━━━  (unread badge: total across all family threads)

  My Families                              (collapsible, unread badge)
  └─ Family X          (2 unread) ▼        (collapsible)
       ● [avatar] userA          · 2m      ← unread: bold + dot
         "hello bạn, gia đình..."
       ● [avatar] Family_Y · userB · 5m
         "cảm ơn bạn nhé"
         [avatar] userC          · 1h      ← read: normal weight
         "nhờ vợ mình nuôi..."
  └─ Family Y          (0 unread) ▼
         [avatar] userD          · 3h
         "cute quá..."

  Sent to Families                         (collapsible)
  └─ [avatar] → Family Z  · 2h
       "cho hỏi về..."
  └─ [avatar] → Family W  · 1d
       "bạn có thể..."

━━ Direct Messages ━━━━━━━━━━━━━━━━━━━━  (unread badge: total)
  ● [avatar] userE     "ok nha bạn..."  · 15m
    [avatar] userF     "thanks nhé..."  · 2h
```

---

## Components

### 1. Conversation List

**Sorting within each section:** unread first, then by last message time desc.

**Each conversation row:**

| Section | Row display |
|---------|-------------|
| My Families → Family X | `[sender avatar]` `sender name` (+ `· family name` if sent on behalf of family) · last message preview · time |
| Sent to Families | `[family avatar]` `→ Family name` · last message preview · time |
| Direct Messages | `[user avatar]` `user name` · last message preview · time |

**Unread indicator:** bold text + filled dot badge on the row; unread count badge on parent section header.

---

### 2. Starting a Thread

**3 entry points:**

| Entry point | Sender options | Receiver |
|-------------|---------------|---------|
| Family Posts → **Message button** | Individual user **or** active family (user chooses) | That family |
| Family Posts → Parents section → **Message icon** next to a user | Individual user only | That specific user |
| My Pets → Parents section → **Message icon** next to a user | Individual user only | That specific user |

**Sender selection (only for family receivers):**
- Show bottom sheet: *"Send as…"*
  - `[user avatar]` Your name (individual)
  - `[family avatar]` Active family name
- For user receivers: no prompt — always individual user as sender

**Visibility rules (hide Message button):**
- Viewing your own family's page → no Message button (can't message your own family)
- Viewing your own profile/user → no Message icon (can't message yourself)

**Thread creation:**
- If a thread already exists between the same sender ↔ receiver → **continue the existing thread**, do not create a new one

---

### 3. Thread View

```
[Header]
  Left: Back button
  Center: [sender avatar] sender name  →  [receiver avatar] receiver name
  Right: [Search icon]  ... (future actions)

[Scrollable messages area]
  ─── Jun 6, 2026 ───
  [userA avatar]
  userA
  hello bạn, gia đình bạn nuôi mèo đẹp nhỉ         [10:00]

  [Family_X badge]  Family_X · userB
  ↩ "hello bạn, gia đình bạn..."                    [10:05]
  cảm ơn bạn nhé

  [Family_X badge]  Family_X · userC
  hehe, nhờ vợ mình nuôi ko đó                      [10:07]

[Fixed bottom bar]
  [Send as: userA ▼]  [Text input]  [Send button]
  └── "Send as" dropdown only shown for family threads
```

**Reply (quoted reply — WhatsApp/Zalo style):**
- Long press a message → shows reply option
- Tapping Reply → input bar shows quoted preview of the selected message
- Sent reply displays the quoted message block above the reply text

**In-thread search:**
- Tap **Search icon** in thread header → search bar slides in below header, messages area shifts down
- **Server-side search** — queries the full thread history, not just loaded messages
- Min 2 characters; debounce 300ms
- Matching messages are highlighted; non-matching messages dimmed
- Up/down arrows to jump between matches; jumps to the correct position in the thread even if message not yet loaded
- Tap × or Back → dismiss search bar, restore normal view

**"Send as" dropdown (family threads only):**
- Options: individual user / active family
- Default: individual user
- Selection persists within the session for this thread

**Tap sender name/family name in header:**
- → Opens a new thread with that entity (if no thread exists yet)
- If receiver is a family: show sender selection prompt
- If receiver is a user: individual sender only

**Thread membership display:**
- On the receiving family side, each reply shows which member sent it: `Family_X · userB`
- On the sending side (individual or family), messages show the sender's name and avatar

---

### 4. Access Rules

| Scenario | Behaviour |
|----------|-----------|
| User is a member of Family F | Threads received by Family F appear under **My Families > Family F** |
| User is removed from Family F | **My Families > Family F** section hidden immediately; all threads received as Family F member no longer accessible |
| User previously sent to Family F (before or after removal) | Thread remains visible under **Sent to Families** — user is sender, not member |
| User continues thread after removal from Family F | Allowed — uses the existing thread in **Sent to Families** |
| User belongs to multiple families | Each family appears as a separate collapsible row under **My Families** |

**Unread flags are per-member:**
- Example: userA messages Family F (members B and C). B reads the message → B's unread flag cleared. C has not read → C still sees unread badge. Independent of each other.

---

## API Endpoints Required

All calls go to `POST /graphql`.

---

### BG. Query: `MessageThreads`

Fetch all threads for the current user (all sections).

**Auth:** Required

```graphql
query MessageThreads {
  messageThreads {
    id
    type           # FAMILY_RECEIVED | FAMILY_SENT | DM
    sender {
      type         # USER | FAMILY
      id
      name
      avatarUrl
    }
    receiver {
      type         # USER | FAMILY
      id
      name
      avatarUrl
    }
    lastMessage {
      body
      sentAt
    }
    unreadCount
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### BH. Query: `ThreadMessages`

Fetch messages in a thread (paginated, **oldest first** — `sentAt asc`). Load earlier messages by passing the oldest known message cursor.

**Auth:** Required

```graphql
query ThreadMessages($threadId: ID!, $cursor: String, $limit: Int) {
  threadMessages(threadId: $threadId, cursor: $cursor, limit: $limit) {
    messages {
      id
      sender {
        userId
        userName
        userAvatarUrl
        familyId
        familyName
      }
      body
      repliedTo {
        id
        body
        sender {
          userName
          familyName
        }
      }
      sentAt
    }
    nextCursor
    hasMore
  }
}
```

**Variables:** `{ "threadId": "thread_001", "limit": 20 }`

**Response `200 OK`:**
```json
{
  "data": {
    "threadMessages": {
      "messages": [
        {
          "id": "msg_001",
          "sender": {
            "userId": "user_001",
            "userName": "Minh Dang",
            "userAvatarUrl": "https://cdn.petapp.com/users/user_001/avatar.jpg",
            "familyId": null,
            "familyName": null
          },
          "body": "hello bạn, gia đình bạn nuôi mèo đẹp nhỉ",
          "repliedTo": null,
          "sentAt": "2026-06-01T09:00:00Z"
        }
      ],
      "nextCursor": "cursor_xyz",
      "hasMore": true
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `THREAD_NOT_FOUND` | Thread does not exist |
| `NOT_THREAD_MEMBER` | Caller is not a member of this thread |

---

### BI. Mutation: `StartThread`

Create or retrieve an existing thread between sender and receiver.

**Auth:** Required

```graphql
mutation StartThread($input: StartThreadInput!) {
  startThread(input: $input) {
    id
    type
    sender { type id name avatarUrl }
    receiver { type id name avatarUrl }
    unreadCount
  }
}
```

**Variables:**
```json
{
  "input": {
    "senderType": "USER",
    "senderId": "user_001",
    "receiverType": "FAMILY",
    "receiverId": "fam_xyz"
  }
}
```

**Note:** If a thread already exists for the same sender ↔ receiver pair, returns the existing thread (no duplicate created).

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `RECEIVER_NOT_FOUND` | Receiver user or family does not exist |

---

### BJ. Mutation: `SendMessage`

Send a message in a thread.

**Auth:** Required

```graphql
mutation SendMessage($threadId: ID!, $input: SendMessageInput!) {
  sendMessage(threadId: $threadId, input: $input) {
    id
    sender {
      userId
      userName
      userAvatarUrl
      familyId
      familyName
    }
    body
    repliedTo {
      id
      body
      sender { userName familyName }
    }
    sentAt
  }
}
```

**Variables:**
```json
{
  "threadId": "thread_001",
  "input": {
    "body": "hello bạn, gia đình bạn nuôi mèo đẹp nhỉ",
    "repliedToId": null
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `THREAD_NOT_FOUND` | Thread does not exist |
| `NOT_THREAD_MEMBER` | Caller is not a member of this thread |
| `MESSAGE_EMPTY` | Message body is blank |

---

### BK. Mutation: `MarkThreadRead`

Mark a thread as read for the current user.

**Auth:** Required

```graphql
mutation MarkThreadRead($threadId: ID!) {
  markThreadRead(threadId: $threadId)
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `THREAD_NOT_FOUND` | Thread does not exist |

---

### BL. Query: `UnreadMessageCount`

For the Messages icon red dot badge in Explore and My Pets headers.

**Auth:** Required

```graphql
query UnreadMessageCount {
  unreadMessageCount
}
```

**Note:** Delivered via **WebSocket push** when available; falls back to polling every 30s. Fetched separately from the thread list (not bundled).

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### BM. Query: `SearchThreadMessages`

Server-side search across all messages in a thread.  
**Auth:** Required

```graphql
query SearchThreadMessages($threadId: ID!, $q: String!, $cursor: String, $limit: Int) {
  searchThreadMessages(threadId: $threadId, q: $q, cursor: $cursor, limit: $limit) {
    messages {
      id
      body
      sentAt
      sender {
        userId
        userName
        familyId
        familyName
      }
    }
    totalCount
    nextCursor
    hasMore
  }
}
```

**Variables:** `{ "threadId": "thread_001", "q": "mèo", "limit": 20 }`

**Notes:**
- Min 2 characters required (enforced client-side before calling)
- Results ordered by `sentAt asc` — client highlights matched messages and scrolls to position

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `THREAD_NOT_FOUND` | Thread does not exist |
| `NOT_THREAD_MEMBER` | Caller is not a member of this thread |
| `QUERY_TOO_SHORT` | Search query is less than 2 characters (server-side enforcement) |

---

## User Flow Diagrams

### Open Messages from header icon

```
User taps Messages icon
  └─> [not authenticated] → redirect to Login
  └─> [authenticated] → load MessageThreads
        └─> Render grouped list (My Families / Sent to Families / DM)
```

### Start thread from Family Posts

```
User taps Message button on Family F
  └─> [not authenticated] → redirect to Login
  └─> [is member of Family F] → Message button hidden (own family)
  └─> [authenticated, non-member]
        └─> Show "Send as…" bottom sheet
              ├─ Send as self → StartThread { senderType=USER, receiverType=FAMILY }
              └─ Send as active family → StartThread { senderType=FAMILY, receiverType=FAMILY }
                    └─> If thread exists → open existing thread
                    └─> If new → create thread → open thread view
```

### Reply in thread

```
User taps Send
  └─> SendMessage { threadId, body, repliedToId (if reply) }
        └─> Message appended to thread
              └─> MarkThreadRead { threadId } called automatically on open
```

---

## Edge Cases

| Case | Behaviour |
|------|-----------|
| User removed from Family F | My Families > Family F hidden; Sent to Families threads unaffected |
| User re-added to Family F | My Families > Family F restored; previous threads visible again |
| Thread exists, user starts new thread with same entity | Returns existing thread — no duplicate |
| Family has 0 unread in My Families | Section still shown (collapsed); only hidden if user has no families at all |
| No conversations yet (new user) | Each section shows empty state: "No conversations yet" |
| Receiver family is deleted | Thread archived — visible as read-only, cannot send new messages |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Unread flags | Per-member, independent (not per-family) |
| 2 | Who can message a family | Any authenticated user — from Family Posts Message button |
| 3 | Sender for family threads | User chooses: individual or active family — prompt shown at thread start |
| 4 | Sender for DM | Always individual user — no prompt |
| 5 | Thread deduplication | Same sender ↔ receiver → reuse existing thread |
| 6 | Access after removal from family | My Families threads hidden; Sent to Families threads remain |
| 7 | Reply style | Quoted reply (WhatsApp/Zalo style) |
| 8 | Start thread from within thread | Tap header name → new thread with that entity |
| 9 | Thread message sort order | Oldest first (`sentAt asc`); load earlier messages by paging up |
| 10 | Unread count delivery | **WebSocket** (push) preferred for real-time dot badge; fall back to polling (every 30s) if WebSocket unavailable |
