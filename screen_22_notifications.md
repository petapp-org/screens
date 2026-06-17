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

Flat list, newest first, infinite scroll. Each row has an icon/avatar, text, optional thumbnail, relative time, and **bold when unread** (no red dot). The header **Notifications icon** is red-dotted when there are any unread items. Tapping a row marks it read (`MarkNotificationAsRead (BT)`) and deep-links to its target.

**Notification categories (`category` enum in contract):**

| Category (enum) | Trigger | Row text (example) | Tap target |
|------------------|---------|--------------------|------------|
| `CARE_REMINDER` | Scheduled care reminder for the user's pet | "Time to care for `{pet}`" + detail snippet | Pet Detail (`screen_9`) |
| `VACCINE_DUE` | Upcoming or overdue vaccine for the user's pet | "Vaccine due for `{pet}`" + detail snippet | Pet Detail (`screen_9`) |
| `HEALTH_SIGNAL` | AI detects a possible health issue in the user's pet media — fired from a user-initiated AI scan (`requestHealthCheck`, screen_7 §3a), **not** automatically on post publish | "AI Health Alert on `{pet}`" + detail snippet | Pet Detail (`screen_9`) |
| `SYSTEM` | System-level announcements or alerts | `title` value | Varies by `data` payload |

> **`UNSPECIFIED`** is a contract sentinel — treat as `SYSTEM` in the UI.

> **Legacy type strings** (`NEW_COMMENT`, `POST_LOVES`, `INVITE_ACCEPTED`, etc.) are **not in the current contract**. These notification types are planned for a future enrichment phase — tracked as target SDL GAPs (see field table below).

**Grouping:** `POST_LOVES`-style grouping (multiple actors per post) is a **GAP** (no `groupCount` in contract yet — see field table). Render each row independently until the backend adds grouping support.

> ⚠️ **`MISSING_NEARBY`/`PET_MISSING`/`PET_FOUND`/`RESCUE_INQUIRY`** notification categories are planned but not yet in the contract enum. They will be added in a later backend enrichment slice.

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
query MyNotifications($after: String) {
  myNotifications(first: 20, after: $after) {
    edges {
      cursor
      node {
        id
        userId
        category   # CARE_REMINDER | VACCINE_DUE | HEALTH_SIGNAL | SYSTEM
        channel    # PUSH | IN_APP | EMAIL
        status
        title
        body
        data
        createdAt
        readAt     # null = unread; non-null DateTime = read
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

> **Field notes:**
>
> | Contract field | Notes |
> |----------------|-------|
> | `category` | Replaces old `type` (see category table above) |
> | `readAt` | Replaces `isRead`; "unread" = `readAt == null` |
> | `title` + `body` | Replaces `preview`; `title` is the bold headline, `body` is the snippet |
> | `data` | `[String!]!` — structured payload for deep-linking (e.g. `["PET", "pet_111"]`) |
> | `actor {}` | ⚠️ **GAP** (not in contract — target SDL ENRICH) |
> | `target {}` | ⚠️ **GAP** (not in contract — target SDL ENRICH) |
> | `thumbnailUrl` | ⚠️ **GAP** (not in contract — target SDL ENRICH) |
> | `groupCount` | ⚠️ **GAP** (not in contract — target SDL ENRICH) |

**Variables:** `{ "after": null }`

**Response `200 OK`:**
```json
{
  "data": {
    "myNotifications": {
      "edges": [
        {
          "cursor": "cursor_noti_001",
          "node": {
            "id": "noti_001",
            "userId": "user_001",
            "category": "HEALTH_SIGNAL",
            "channel": "IN_APP",
            "status": "SENT",
            "title": "AI Health Alert on Pudding",
            "body": "Possible skin irritation detected in a recent photo",
            "data": ["PET", "pet_222"],
            "createdAt": "2026-06-07T08:55:00Z",
            "readAt": null
          }
        },
        {
          "cursor": "cursor_noti_002",
          "node": {
            "id": "noti_002",
            "userId": "user_001",
            "category": "CARE_REMINDER",
            "channel": "IN_APP",
            "status": "SENT",
            "title": "Time to groom Bụi",
            "body": "Weekly grooming reminder",
            "data": ["PET", "pet_111"],
            "createdAt": "2026-06-07T07:10:00Z",
            "readAt": "2026-06-07T07:15:00Z"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": true,
        "endCursor": "cursor_noti_002"
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |

---

### BT. Mutation: `MarkNotificationAsRead` / `MarkAllNotificationsAsRead`

Mark one notification as read, or mark all as read.

**Auth:** Required

**Mark single (tap a row):**
```graphql
mutation MarkNotificationAsRead($notificationId: ID!) {
  markNotificationAsRead(notificationId: $notificationId) {
    id
    readAt
  }
}
```

**Variables:** `{ "notificationId": "noti_001" }`

**Mark all (tap "Mark all as read"):**
```graphql
mutation MarkAllNotificationsAsRead {
  markAllNotificationsAsRead   # returns Int! — count of notifications marked read
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `NOTIFICATION_NOT_FOUND` | `notificationId` does not exist for this user |

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
  └─> [authenticated] → MyNotifications (BS) { first: 20, after: null } → activity feed
```

### Infinite Scroll

```
User scrolls to bottom
  └─> MyNotifications (BS) { first: 20, after: pageInfo.endCursor }
        └─> Append new rows
              └─> pageInfo.hasNextPage=false → no more rows
```

### Open a notification

```
User taps a notification row
  └─> MarkNotificationAsRead (BT) { notificationId }  → readAt set; row un-bolds; Notifications-icon dot decrements
        └─> Deep-link via data[] payload (Pet Detail / etc.)
```

---

## Edge Cases

| Case | Behaviour |
|------|-----------|
| Deep-link target deleted (pet removed, etc.) | Tapping marks it read via `MarkNotificationAsRead` and shows "This content is no longer available" instead of deep-linking |
| `readAt != null` | Row displayed in normal weight (not bold) |
| `readAt == null` | Row displayed in **bold** (unread) |
| `POST_LOVES`-style grouping | **GAP** — `groupCount` not in contract yet; render each notification independently until backend enrichment |
| Deep-link interpretation | Parse `data[]` array to determine target: `["PET", "<id>"]` → Pet Detail, `["CARE", "<id>"]` → Pet Detail care tab, etc. |
| New user, no activity | "No notifications yet" |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Screen split | Split from the old two-tab screen: this screen is the **general activity feed only**; messaging lives in **Messages (`screen_10`)** |
| 2 | Header entry point | **Notifications icon** `🔔` (Explore + My Pets + More headers); red dot (no number) when any activity unread (`UnreadNotificationCount BU`). Separate from the **Messages icon** `✉` (`screen_10`) |
| 3 | Unread indicator | **Inside the screen**: bold only, no red dot. **Header Notifications icon**: red dot when any unread |
| 4 | Parent invite received | Excluded — only `INVITE_ACCEPTED` notified (future `SYSTEM` notification via `data` payload) |
| 5 | Loves grouping | **GAP** — `groupCount` not in contract; render flat until enrichment |
| 6 | `MISSING_NEARBY` deep-link | **GAP** — `target {}` not in contract; parse `data[]` when enrichment ships |
| 7 | Unread count delivery | WebSocket push preferred; poll every 30s fallback |
| 8 | `isRead` field | Removed — use `readAt != null` to determine read state |
| 9 | `markNotificationRead(id: null)` | Removed — use `markAllNotificationsAsRead` mutation instead |
