# Screen 10: Notifications

## Overview

Notification center with **two tabs**: **Chats** and **Notifications**.
- **Chats** — message notifications: a flat list of conversation threads relevant to the user *right now*, scoped to the user's **currently active family** (plus the user's own DMs and threads the user personally sent).
- **Notifications** — general activity notifications (comments, replies, follows, loves, AI health alerts, missing/found pets, invite accepted, etc.).

Auth required. Accessible from: the **Messages icon** in the Explore header (screen_1) and My Pets header (screen_8).
The Messages icon shows a **red dot** (no number) whenever there is any unread — combined across both tabs (chats unread + general unread). Inside the screen there is **no red dot**; unread items are shown in **bold** instead.

> **Terminology — "active family":** A user can belong to multiple families but operates as exactly **one active family at a time** (set/switched via `SetActiveFamily (AG)` in **Profile Settings, screen_5** → Family Pages). The active family is the pivot for the Chats tab: it determines which family-received threads are visible and which messages count as unread. See **Access & Active-Family Rules** below.

---

## UI Layout

```
[Header]
  Title: "Notifications"

[Tab bar]
  [ Chats ]   [ Notifications ]      ← tab label is BOLD when that tab has unread (no badge, no dot)
  ──────                              (active tab underlined)

────────────────────────────────────────────────────────────
TAB 1 — CHATS  (flat thread list, scoped to active family)
              scope: Pudding's Family   ← current active family, shown once at top
────────────────────────────────────────────────────────────

  ● [avatarA] userA                              · 2m   ← DM: name only, no arrow (unread: bold)
    "hello bạn, gia đình bạn nuôi mèo đẹp nhỉ"

  ● [avatarB] userB → Pudding's Family           · 5m   ← user → my active family
    "nhà mình muốn hỏi..."

  ● [avatarFamY] Family_Y · userC → Pudding's…   · 1h   ← family → my active family (read: normal)
    "cảm ơn nhé"

    [avatarFamZ] you → Family Z                  · 2h   ← a thread I sent to a family
    "cho hỏi về bé mèo nhà mình..."

  (Empty: "No conversations yet")

────────────────────────────────────────────────────────────
TAB 2 — NOTIFICATIONS  (general activity, newest first)
────────────────────────────────────────────────────────────

  ● [avatarD] Linh Pham commented on your post          · 3m
    "bé cún xinh quá ❤️"
  ● [⚠ icon] AI Health Alert on Pudding                 · 1h
    "Possible skin irritation detected in a recent photo"
  ● [avatarE] Minh and 4 others loved your post         · 2h
    [thumbnail]
    [avatarF] Hoa started following your family          · 5h
    [📍 icon] A missing dog was reported near you         · 1d

  (Empty: "No notifications yet")
```

---

## Components

### TAB 1 — Chats

#### 1. Chats Thread List

A **single flat list** — no "My Families / Sent to Families / DM" sections. The list contains exactly these threads (see **Access & Active-Family Rules**):

1. Threads **received by the user's currently active family** (inbound from an external user or an external family).
2. The user's **DMs** (both sent and received).
3. Threads the user **personally sent** to other families.

Threads received by the user's **non-active** families are **not shown** at all.

**Scope line:** The current **active family** name is shown once at the top of the list (e.g. *"Pudding's Family"*) — it is the receiver of every type-2 row below, so it is not repeated as a separate header.

**Sorting:** unread first, then by last message time desc.

**Row rendering — hybrid by type** (DM compact; family threads spell out the receiver):

| Type | Avatar | Title format | Example |
|------|--------|--------------|---------|
| **DM** | sender's user avatar | `{userName}` — **name only, no arrow** | `userA` |
| **To active family** (individual sender) | sender's user avatar | `{userName} → {myActiveFamily}` | `userB → Pudding's Family` |
| **To active family** (family sender) | sender family avatar | `{senderFamily} · {member} → {myActiveFamily}` | `Family_Y · userC → Pudding's Family` |
| **I sent to a family** | receiver family avatar | `you → {receiverFamily}` | `you → Family Z` |

- `{myActiveFamily}` = the user's current active family name (constant across all type-2 rows; client fills it from the active-family context). Truncate with `…` if long.
- **Reading the type:** no arrow → **DM**; `… → {my active family}` → **received by my active family**; `you → …` → **I sent it**.

**Other row elements:**

| Element | Display |
|---------|---------|
| Last message preview | One line, truncated. May be my own message (rendered plainly — never bold; no special "You:" label). |
| Time | Relative time of last message. |
| CHARITY badge | Small badge next to a family name when that family `type = charity`. |

**Unread indicator:** **bold** row text only. **No red dot** inside the screen (bold already conveys unread). The **Chats tab label** is bold when there are any unread threads.

**Row tap:** → open **Thread View** (see below); calls `MarkThreadRead (BK)` on open.

> A message preview can be the user's own last reply (e.g. in a DM the user answered). That row is **never bold** — outbound messages never mark a thread unread.

---

### TAB 2 — Notifications

#### 2. General Notification List

Flat list, newest first, infinite scroll. Each row has an icon/avatar, text, optional thumbnail, relative time, and **bold when unread** (no red dot). The **Notifications tab label** is bold when there are any unread items. Tapping a row marks it read (`MarkNotificationRead (BT)`) and deep-links to its target.

**Notification types:**

| Type (enum) | Trigger | Row text (example) | Tap target |
|-------------|---------|--------------------|------------|
| `NEW_COMMENT` | Someone comments on the user's post | "`{actor}` commented on your post" + comment snippet | Post Detail (scrolled to comment) |
| `NEW_REPLY` | Someone replies to the user's comment | "`{actor}` replied to your comment" + reply snippet | Post Detail (scrolled to reply) |
| `POST_LOVES` | People love the user's post (**grouped**) | "`{actor}` and `{N}` others loved your post" + thumbnail | Post Detail |
| `FAMILY_NEW_POST` | A family the user **follows** publishes a new post | "`{family}` shared a new post" + thumbnail | Post Detail |
| `NEW_FOLLOWER` | Someone follows the user's family | "`{actor}` started following your family" | The follower's User Posts (or their family) |
| `INVITE_ACCEPTED` | A parent the user invited accepts the family invite | "`{actor}` accepted your invite to `{family}`" | Family Posts / My Pets parents |
| `HEALTH_ALERT` | AI detects a possible health issue in the user's pet media | "AI Health Alert on `{pet}`" + detail snippet | Pet Detail (screen_9) |
| `MISSING_NEARBY` | A pet is reported missing near the user's location | "A missing `{species}` was reported near you" | Missing pet detail (Coming Soon) |
| `PET_MISSING` | The user's own pet is marked missing (status confirmation) | "`{pet}` was marked as missing" | Pet Detail (screen_9) |
| `PET_FOUND` | A previously-missing pet (own, or one the user reported/follows) is marked found | "`{pet}` was marked as found" | Pet Detail (screen_9) |

**Grouping:** `POST_LOVES` collapses multiple lovers of the **same post** into one row (`{actor} and {N} others`). Other types are one row per event.

> **Not included:** "Parent invite **received**" is intentionally **not** a notification type — there's no reliable way to surface an incoming invite as a noti in this model. Only `INVITE_ACCEPTED` (the outbound invite being accepted) is notified.

---

### 3. Starting a Thread

*(composing happens from other screens, not from this inbox)*

**3 entry points:**

| Entry point | Sender options | Receiver |
|-------------|---------------|---------|
| Family Posts → **Message button** | Individual user **or** active family (user chooses) | That family |
| Family Posts → Parents section → **Message icon** next to a user | Individual user only | That specific user |
| My Pets → Parents section → **Message icon** next to a user | Individual user only | That specific user |

**Sender selection (only for family receivers):**
- Bottom sheet *"Send as…"*:
  - `[user avatar]` Your name (individual)
  - `[family avatar]` Active family name
- For user receivers: no prompt — always individual user as sender.

**Visibility rules (hide Message button / icon):**
- Viewing a family **you are a member of** (active **or** non-active) → no Message button (you're already on the receiving side; can't message your own family).
- Viewing your own profile/user → no Message icon (can't message yourself).

**Thread creation:** If a thread already exists for the same sender ↔ receiver → **continue the existing thread**; never duplicate.

---

### 4. Thread View

```
[Header]
  Left: Back button
  Center: [sender avatar] sender name  →  [receiver avatar] receiver name
  Right: [Search icon]

[Scrollable messages area — oldest first, auto-scroll to bottom on open]
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
- Long press a message → Reply option.
- Tapping Reply → input bar shows quoted preview of the selected message.
- Sent reply displays the quoted block above the reply text.

**In-thread search:**
- Tap **Search icon** → search bar slides in below header.
- **Server-side search** (`SearchThreadMessages (BM)`) — queries full thread history.
- Min 2 characters; debounce 300ms.
- Matching messages highlighted, non-matching dimmed; up/down arrows jump between matches (loads position even if not yet paged in).
- Tap × / Back → restore normal view.

**"Send as" dropdown (family threads only):** individual user / active family; default individual; persists within the session for this thread.

**Tap sender/receiver name in header:** opens (or reuses) a thread with that entity; if receiver is a family, show sender-selection prompt.

**Thread membership display:** on the receiving family side each reply shows which member sent it (`Family_X · userB`); on the sending side, sender's name + avatar.

---

## Access & Active-Family Rules

The Chats tab is **scoped to the user's currently active family**. These rules are the source of truth.

### Thread visibility

| Thread kind | Visible in Chats? |
|-------------|-------------------|
| Received by the user's **active** family (inbound) | ✅ Yes |
| Received by the user's **non-active** family | ❌ No — hidden entirely |
| **DM** to/from the user | ✅ Yes — always (not scoped to active family) |
| Thread the user **personally sent** to a family | ✅ Yes — always (user is the sender) |

When the user **switches active family** (F → M, via `SetActiveFamily (AG)` in Profile Settings):
- Threads received by **F disappear** from Chats; threads received by **M appear**.
- DMs and the user's own sent threads are unaffected (always visible).

### Unread rules

- **Direction:** Only **inbound** messages can mark a thread unread. A reply the user sends — or a reply a **co-member** of the user's family sends (which is outbound *from* that family toward the original sender) — never marks the thread unread for the user or for other co-members. Example: A and B are members of F; C messages F → both A and B get unread. A replies → B does **not** get a new unread (A's reply is outbound from F to C).
- **Activation timestamp (family-received threads):** A family-received message counts as **unread only if** `message.sentAt > the moment that family became active for this user`. Messages that arrived while the family was **not** active (or **before** the user joined the family) are treated as already-read and are **never** marked unread when the user later activates that family.
  - **New member:** is invited to F → F's pre-existing messages predate the user's activation → **no unread** shown to the new member.
  - **Switch F → M:** only messages sent **after** the switch-to-M count as unread; older messages in M's threads stay read.
- **DMs & own sent threads:** standard unread (unread if there are inbound messages the user hasn't read); not gated by any active-family timestamp.

### Removal from a family

- When the user is **removed** from a family, they **lose all access** to that family's received threads **and the entire chat-notification history** for that family. Nothing from that family remains visible.
- If the removed family was the user's **active** family, the server **auto-switches** the active family to another family the user belongs to — applying to **both** the Chats noti scope **and** the action/context everywhere (My Pets, "Send as", etc.). The newly active family's received threads appear immediately.
- **Edge — no other family:** if the removed family was the only one, active family becomes **unset**; the Chats tab then shows only the user's DMs and own sent threads.

---

## API Endpoints Required

All calls go to `POST /graphql`.

---

### BG. Query: `MessageThreads`

Fetch the Chats list for the current user. The server returns **only** the threads visible per the **Access & Active-Family Rules** (active-family-received + DMs + own-sent); non-active-family threads are excluded server-side. `unreadCount` is computed using the activation-timestamp rule.

**Auth:** Required

```graphql
query MessageThreads {
  messageThreads {
    id
    type             # FAMILY_RECEIVED | DM | FAMILY_SENT
    counterpart {    # the "other side" to render on the row
      type           # USER | FAMILY
      id
      name
      avatarUrl
      isCharity      # true → show CHARITY badge (family only)
    }
    direction        # INBOUND (received) | OUTBOUND (I sent it)
    lastMessage {
      body
      sentAt
      senderType       # USER | FAMILY
      senderName       # the member who actually sent it
      senderFamilyName # set when senderType = FAMILY (sent on behalf of a family); else null
    }
    unreadCount        # 0 when read; honors direction + activation-timestamp rules
  }
}
```

**Row rendering from this shape** (hybrid — DM compact, family threads spell out the receiver):
- `type = DM` → title `{counterpart.name}` (name only, no arrow), avatar = `counterpart.avatarUrl`.
- `type = FAMILY_RECEIVED`, `lastMessage.senderType = USER` → title `{lastMessage.senderName} → {myActiveFamily}`, avatar = `counterpart.avatarUrl`.
- `type = FAMILY_RECEIVED`, `lastMessage.senderType = FAMILY` → title `{lastMessage.senderFamilyName} · {lastMessage.senderName} → {myActiveFamily}`, avatar = `counterpart.avatarUrl`.
- `type = FAMILY_SENT` → title `you → {counterpart.name}`, avatar = `counterpart.avatarUrl`.
- `{myActiveFamily}` is supplied by the client from the active-family context (not in this response — it is the same family for every `FAMILY_RECEIVED` row).
- `counterpart.isCharity = true` → show CHARITY badge next to the family name.
- `unreadCount > 0` → row bold.

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
    counterpart { type id name avatarUrl isCharity }
    direction
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

Chats-tab unread count — `> 0` bolds the Chats tab label and contributes to the **Messages-icon red dot** (combined with `NotificationUnreadCount (BU)`).

**Auth:** Required

```graphql
query UnreadMessageCount {
  unreadMessageCount
}
```

**Note:** Delivered via **WebSocket push** when available; falls back to polling every 30s. Honors the active-family scope + activation-timestamp rules (only currently-visible, post-activation inbound threads count).

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
- Min 2 characters required (enforced client-side before calling).
- Results ordered by `sentAt asc` — client highlights matched messages and scrolls to position.

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `THREAD_NOT_FOUND` | Thread does not exist |
| `NOT_THREAD_MEMBER` | Caller is not a member of this thread |
| `QUERY_TOO_SHORT` | Search query is less than 2 characters (server-side enforcement) |

---

### BS. Query: `Notifications`

General-notification feed for the Notifications tab (paginated, newest first).

**Auth:** Required

```graphql
query Notifications($cursor: String, $limit: Int) {
  notifications(cursor: $cursor, limit: $limit) {
    items {
      id
      type          # NEW_COMMENT | NEW_REPLY | POST_LOVES | FAMILY_NEW_POST |
                    # NEW_FOLLOWER | INVITE_ACCEPTED | HEALTH_ALERT |
                    # MISSING_NEARBY | PET_MISSING | PET_FOUND
      actor {       # who triggered it; null for system/AI events
        id
        name
        avatarUrl
      }
      target {      # what the row deep-links to
        type        # POST | COMMENT | PET | FAMILY | USER | MISSING_REPORT
        id
      }
      preview       # snippet/text shown under the title (nullable)
      thumbnailUrl  # media thumbnail (POST_LOVES / FAMILY_NEW_POST); else null
      groupCount    # extra count for grouped types (POST_LOVES); else null
      isRead
      createdAt
    }
    nextCursor
    hasMore
  }
}
```

**Variables:** `{ "cursor": null, "limit": 20 }`

**Response `200 OK`:**
```json
{
  "data": {
    "notifications": {
      "items": [
        {
          "id": "noti_001",
          "type": "NEW_COMMENT",
          "actor": {
            "id": "user_044",
            "name": "Linh Pham",
            "avatarUrl": "https://cdn.petapp.com/users/user_044/avatar.jpg"
          },
          "target": { "type": "POST", "id": "post_988" },
          "preview": "bé cún xinh quá ❤️",
          "thumbnailUrl": null,
          "groupCount": null,
          "isRead": false,
          "createdAt": "2026-06-07T08:55:00Z"
        },
        {
          "id": "noti_002",
          "type": "POST_LOVES",
          "actor": {
            "id": "user_071",
            "name": "Minh",
            "avatarUrl": "https://cdn.petapp.com/users/user_071/avatar.jpg"
          },
          "target": { "type": "POST", "id": "post_512" },
          "preview": null,
          "thumbnailUrl": "https://cdn.petapp.com/posts/post_512/thumb.jpg",
          "groupCount": 4,
          "isRead": false,
          "createdAt": "2026-06-07T07:10:00Z"
        }
      ],
      "nextCursor": "cursor_abc",
      "hasMore": true
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### BT. Mutation: `MarkNotificationRead`

Mark one notification (or all) as read.

**Auth:** Required

```graphql
mutation MarkNotificationRead($id: ID) {
  markNotificationRead(id: $id)   # id null → mark ALL general notifications read
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `NOTIFICATION_NOT_FOUND` | `id` provided but does not exist for this user |

---

### BU. Query: `NotificationUnreadCount`

Notifications-tab unread count — `> 0` bolds the Notifications tab label and contributes to the **Messages-icon red dot** (combined with `UnreadMessageCount (BL)`).

**Auth:** Required

```graphql
query NotificationUnreadCount {
  notificationUnreadCount
}
```

**Note:** Delivered via **WebSocket push** when available; falls back to polling every 30s.

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

## User Flow Diagrams

### Open Notifications from Messages icon

```
User taps Messages icon (Explore / My Pets header)
  └─> [not authenticated] → redirect to Login
  └─> [authenticated] → open screen on last-used tab (default: Chats)
        ├─ Chats tab        → MessageThreads (BG)   → flat thread list
        └─ Notifications tab → Notifications (BS)    → general noti list
```

### Switch active family while Chats is open

```
User switches active family F → M (SetActiveFamily AG, Profile Settings)
  └─> Chats re-fetches MessageThreads (BG)
        └─> F-received threads drop out; M-received threads appear
              └─> DMs + own-sent threads unchanged
                    └─> unread recomputed (only post-switch messages unread)
```

### Removed from active family

```
Server: user removed from active family F
  └─> F-received threads + chat-noti history for F purged from view
        └─> active family auto-switches F → M (noti scope + action context)
              ├─ has other family (M) → M-received threads appear; M now active everywhere
              └─ no other family       → active = unset; Chats shows DMs + own-sent only
```

### Open a thread (Chats)

```
User taps a thread row
  └─> ThreadMessages (BH) { threadId, limit: 20 }  (oldest first, auto-scroll to bottom)
        └─> MarkThreadRead (BK) { threadId }  → row un-bolds; badges decrement
```

### Open a general notification

```
User taps a notification row
  └─> MarkNotificationRead (BT) { id }  → row un-bolds; tab badge decrements
        └─> Deep-link to target (Post Detail / Pet Detail / User Posts / etc.)
```

### Reply in thread

```
User taps Send
  └─> SendMessage (BJ) { threadId, body, repliedToId (if reply) }
        └─> Message appended (outbound — never marks thread unread for anyone)
```

---

## Edge Cases

| Case | Behaviour |
|------|-----------|
| Switch active family F → M | F-received threads hidden; M-received shown; DMs + own-sent unchanged |
| New member invited to family | No unread for messages predating activation; old threads show as read |
| Removed from non-active family | That family's received threads + chat-noti history removed; active family unchanged |
| Removed from active family (has other family) | Auto-switch active → another family; its threads appear; context switches everywhere |
| Removed from active family (no other family) | Active = unset; Chats shows only DMs + own-sent threads |
| Co-member replies in a family-received thread | Outbound from the family → does **not** mark unread for other co-members; only the original external sender is notified |
| User's own message is the last in a thread | Row shows the preview plainly; never bold (outbound never unread) |
| Thread exists, user starts new thread with same entity | Returns existing thread — no duplicate |
| Receiver family is deleted | Thread archived — read-only, cannot send new messages |
| General notification target deleted (post/comment/pet removed) | Tapping marks it read and shows a "This content is no longer available" state instead of deep-linking |
| `POST_LOVES` for the same post by many users | Collapsed to one grouped row (`{actor} and {N} others`) |
| Parent invite received | **Not** a notification — only `INVITE_ACCEPTED` is surfaced |
| New user, no activity | Chats: "No conversations yet"; Notifications: "No notifications yet" |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Screen identity | Renamed "Messages" → **"Notifications"** with two tabs: **Chats** and **Notifications** |
| 2 | Header entry point | Existing **Messages icon** (Explore + My Pets headers); shows a **red dot** (no number) when any unread exists (combined chats + general) |
| 3 | Chats list structure | **Single flat list** — no "My Families / Sent to Families / DM" sections; active family name shown once at top as scope line |
| 3a | Chats row rendering | **Hybrid by type**: DM = name only (no arrow); to active family = `sender → {activeFamily}`; sent = `you → {family}`. Type is read from the arrow pattern |
| 4 | Chats scope | Scoped to **active family** (received) + user's **DMs** + user's **own-sent** threads; non-active-family received threads hidden |
| 5 | Unread indicator | **Inside the screen**: bold only (rows + tab labels), no red dot. **Header Messages icon**: red dot when any unread |
| 6 | Unread direction | Only **inbound** messages mark unread; outbound (own or co-member's family reply) never does |
| 7 | Unread timing | Family-received message is unread only if `sentAt > active-family activation timestamp` for the user |
| 8 | New member unread | None for pre-membership/pre-activation messages |
| 9 | Removal from family | Lose all received threads + chat-noti history for that family |
| 10 | Removal from active family | Auto-switch active family (noti scope + action context); unset if no other family |
| 11 | "You:" notification rows | Removed — sent messages never create a standalone notification; previews may still show own last message plainly |
| 12 | Thread message sort | Oldest first (`sentAt asc`); auto-scroll to bottom on open; page up for older |
| 13 | General noti: parent invite received | Excluded — only `INVITE_ACCEPTED` notified |
| 14 | General noti: loves | Grouped per post (`{actor} and {N} others`) |
| 15 | Unread count delivery | WebSocket push preferred; poll every 30s fallback (both `BL` and `BU`) |
| 16 | CHARITY badge | Shown next to charity family names in Chats rows |
