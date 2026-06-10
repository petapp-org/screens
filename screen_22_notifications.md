# Screen 22: Notifications

## Overview

The **general activity notification center** — a single flat feed of activity relevant to the user: comments, replies, follows, loves, AI health alerts, missing/found pets, invite accepted, etc.

> **Split note:** This screen was split out from the old two-tab "Notifications" screen. **Messaging** (the Chats inbox + Thread View) now lives in **Messages (`screen_10`)**; this screen holds only the **general activity feed**. See `screen_10` Decisions Log #1.

Auth required. Accessible from the **Notifications icon** (`🔔`) in the Explore header (`screen_1`), My Pets header (`screen_8`), and More header (`screen_17`).
The Notifications icon shows a **red dot** (no number) whenever there is any unread activity — driven by `UnreadNotificationCount (BU)`. (The separate **Messages icon** `✉` carries its own red dot from `UnreadMessageCount (BL)`, `screen_10`.) Inside the screen there is **no red dot**; unread items are shown in **bold** instead.

---

## UI Layout

```
[Header]
  Left: Back button
  Title: "Notifications"

────────────────────────────────────────────────────────────
ACTIVITY FEED  (general activity, newest first, infinite scroll)
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

### 1. Activity Feed

Flat list, newest first, infinite scroll. Each row has an icon/avatar, text, optional thumbnail, relative time, and **bold when unread** (no red dot). The header **Notifications icon** is red-dotted when there are any unread items. Tapping a row marks it read (`MarkNotificationRead (BT)`) and deep-links to its target.

**Notification types:**

| Type (enum) | Trigger | Row text (example) | Tap target |
|-------------|---------|--------------------|------------|
| `NEW_COMMENT` | Someone comments on the user's post | "`{actor}` commented on your post" + comment snippet | Post Detail (scrolled to comment) |
| `NEW_REPLY` | Someone replies to the user's comment | "`{actor}` replied to your comment" + reply snippet | Post Detail (scrolled to reply) |
| `POST_LOVES` | People love the user's post (**grouped**) | "`{actor}` and `{N}` others loved your post" + thumbnail | Post Detail |
| `FAMILY_NEW_POST` | A family the user **follows** publishes a new post | "`{family}` shared a new post" + thumbnail | Post Detail |
| `NEW_FOLLOWER` | Someone follows the user's family | "`{actor}` started following your family" | The follower's User Posts (or their family) |
| `INVITE_ACCEPTED` | A parent the user invited accepts the family invite | "`{actor}` accepted your invite to `{family}`" | Family Posts / My Pets parents |
| `HEALTH_ALERT` | AI detects a possible health issue in the user's pet media | "AI Health Alert on `{pet}`" + detail snippet | Pet Detail (`screen_9`) |
| `MISSING_NEARBY` | A pet is reported missing near the user's location | "A missing `{species}` was reported near you" | Lost Pet Detail (`screen_19`), via `target.type = MISSING_REPORT` |
| `PET_MISSING` | The user's own pet is marked missing (status confirmation) | "`{pet}` was marked as missing" | Pet Detail (`screen_9`) |
| `PET_FOUND` | A previously-missing pet (own, or one the user reported/follows) is marked found | "`{pet}` was marked as found" | Pet Detail (`screen_9`) |
| `RESCUE_INQUIRY` | Someone taps **Inquire to Adopt** on a rescue listing the user's **charity family** posted | "`{actor}` is interested in adopting `{petName}`" | Rescue Detail (`screen_26`), via `target.type = RESCUE_LISTING` |

**Grouping:** `POST_LOVES` collapses multiple lovers of the **same post** into one row (`{actor} and {N} others`). Other types are one row per event.

> **`RESCUE_INQUIRY`** is sent to the **charity family's members** (like other family-directed events) and fires only on a user's **first** inquiry for that listing (idempotent — re-inquiring doesn't re-notify). The adoption conversation itself lands in **Messages** (`screen_10`); this notification is the heads-up. Tapping opens the Rescue Detail in manage context.

> **Not included:** "Parent invite **received**" is intentionally **not** a notification type — there's no reliable way to surface an incoming invite as a noti in this model. Only `INVITE_ACCEPTED` (the outbound invite being accepted) is notified.

---

## API Endpoints Required

All calls go to `POST /graphql`.

---

### BS. Query: `MyNotifications`

General-notification feed (paginated, newest first).

**Auth:** Required

```graphql
query MyNotifications($cursor: String, $limit: Int = 50) {
  myNotifications(limit: $limit, cursor: $cursor) {
    notifications {
      id
      type          # NEW_COMMENT | NEW_REPLY | POST_LOVES | FAMILY_NEW_POST |
                    # NEW_FOLLOWER | INVITE_ACCEPTED | HEALTH_ALERT |
                    # MISSING_NEARBY | PET_MISSING | PET_FOUND | RESCUE_INQUIRY
      actor {       # who triggered it; null for system/AI events
        id
        name
        avatarUrl
      }
      target {      # what the row deep-links to
        type        # POST | COMMENT | PET | FAMILY | USER | MISSING_REPORT | RESCUE_LISTING
        id
      }
      preview       # snippet/text shown under the title (nullable)
      thumbnailUrl  # media thumbnail (POST_LOVES / FAMILY_NEW_POST); else null
      groupCount    # extra count for grouped types (POST_LOVES); else null
      isRead
      createdAt
    }
    nextCursor
  }
}
```

**Variables:** `{ "cursor": null, "limit": 20 }`

**Response `200 OK`:**
```json
{
  "data": {
    "myNotifications": {
      "notifications": [
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
      "nextCursor": "cursor_abc"
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

### BU. Query: `UnreadNotificationCount`

Activity unread count — `> 0` drives the **Notifications-icon red dot** (`🔔`, header). Independent of the Messages-icon dot (`UnreadMessageCount (BL)`, `screen_10`).

**Auth:** Required

```graphql
query UnreadNotificationCount {
  unreadNotificationCount
}
```

**Note:** Delivered via **WebSocket push** when available; falls back to polling every 30s.

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

## User Flow Diagrams

### Open Notifications from the Notifications icon

```
User taps Notifications icon (🔔 — Explore / My Pets / More header)
  └─> [not authenticated] → redirect to Login
  └─> [authenticated] → MyNotifications (BS) → activity feed
```

### Open a notification

```
User taps a notification row
  └─> MarkNotificationRead (BT) { id }  → row un-bolds; Notifications-icon dot decrements
        └─> Deep-link to target (Post Detail / Pet Detail / User Posts / Lost Pet Detail / etc.)
```

---

## Edge Cases

| Case | Behaviour |
|------|-----------|
| General notification target deleted (post/comment/pet removed) | Tapping marks it read and shows a "This content is no longer available" state instead of deep-linking |
| `POST_LOVES` for the same post by many users | Collapsed to one grouped row (`{actor} and {N} others`) |
| `MISSING_NEARBY` tapped | Opens **Lost Pet Detail** (`screen_19`) for the `MISSING_REPORT` target |
| Parent invite received | **Not** a notification — only `INVITE_ACCEPTED` is surfaced |
| New user, no activity | "No notifications yet" |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Screen split | Split from the old two-tab screen: this screen is the **general activity feed only**; messaging lives in **Messages (`screen_10`)** |
| 2 | Header entry point | **Notifications icon** `🔔` (Explore + My Pets + More headers); red dot (no number) when any activity unread (`UnreadNotificationCount BU`). Separate from the **Messages icon** `✉` (`screen_10`) |
| 3 | Unread indicator | **Inside the screen**: bold only, no red dot. **Header Notifications icon**: red dot when any unread |
| 4 | Parent invite received | Excluded — only `INVITE_ACCEPTED` notified |
| 5 | Loves grouping | Grouped per post (`{actor} and {N} others`) |
| 6 | `MISSING_NEARBY` target | Deep-links to **Lost Pet Detail** (`screen_19`) via `target.type = MISSING_REPORT` |
| 7 | Unread count delivery | WebSocket push preferred; poll every 30s fallback |
