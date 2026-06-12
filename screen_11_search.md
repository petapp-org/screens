# Screen 11: Search

## Overview

Search for families, users, and pets across the app.  
Auth required — unauthenticated users tapping the Search icon are redirected to Login.  
Accessible from: Search icon in Explore header (screen_1) and My Pets header (screen_8).

---

## UI Layout

```
[Header]
  Left: Back button
  Center: [Search input — auto-focused on open]
    Placeholder: "Search families, users, pets..."

[Filter Tabs]
  Families | Users | Pets

[Results area]
  (empty state on first open — no query yet)
```

---

## Behaviour

- Auto-focuses the input on screen open
- Min **2 characters** to trigger search; debounce **300ms** after last keystroke
- Results update in real-time as user types
- Switching tabs re-runs the same query against the new scope (no extra tap needed)
- Clear (×) button appears in input when query is non-empty → clears input and resets results

---

## Tabs & Search Criteria

### Families tab

Search by: `name`, `@tag`

**Each result row:**
```
[Family avatar]  Family name
                 @tag  ·  city, country
                 N followers  ·  [CHARITY badge if type=charity]
                 [Follow button]  [Donate button — charity only]
```

> followersCount trả số thô; client tự format "3.6k" theo locale (bỏ followerCountDisplay — i18n client-side).

- Tap row → **Family Posts screen** (or My Pets if own family)
- Follow button: same behaviour as Suggested Families widget (screen_1)
- **Unfollow Undo**: tapping "Following" → unfollow optimistically + show 5s toast *"Unfollowed [Family Name]"* + **[Undo]** → if Undo tapped: re-follow silently
- Donate button: charity families only; same behaviour as Suggested Families widget (screen_1)

---

### Users tab

Search by: `display name`, `@username`

**Each result row:**
```
[User avatar]  Display name
               @username
```

- Tap row → **User Posts screen**
- No action buttons on the row

---

### Pets tab

Search by: `breed`, `species`

**Each result row:**
```
[Pet avatar]  Pet name
              breed  ·  species
              [Family avatar]  Family name   ← the family this pet belongs to
```

- Only shows pets where `isPublic = true` AND the pet has at least 1 post visible to the viewer (server-side filtered)
- Tap row → **Pet Posts screen** for that pet

---

## Empty & Loading States

| State | Display |
|-------|---------|
| First open (no query) | Blank results area — no prompt needed |
| Query < 2 chars | Blank results area — no error shown |
| Loading (debounce in progress) | Subtle loading indicator below tabs |
| No results found | "No [families / users / pets] found for "[query]"" |
| Query cleared | Reset to blank results area |

---

## API Endpoints Required

All calls go to `POST /graphql`.

---

### BN. Query: `SearchFamilies`

```graphql
query SearchFamilies($q: String!, $first: Int! = 20, $after: String) {
  searchFamilies(q: $q, first: $first, after: $after) {
    edges {
      cursor
      score         # Float! — relevance score (ADR-0026); node is nullable (null if index stale / entity deleted)
      node {
        id
        name
        tag
        avatarUrl
        familyType
        location {
          city
          cityCode
          country
          countryCode
        }
        social {
          followersCount
          isFollowedByMe
        }
      }
    }
    pageInfo {
      endCursor
      hasNextPage
    }
  }
}
```

**Variables:** `{ "q": "pudding", "first": 20 }`

**Errors:**

| Code | Scenario |
|------|----------|
| `QUERY_TOO_SHORT` | Query is less than 2 characters |

---

### BO. Query: `SearchUsers`

```graphql
query SearchUsers($q: String!, $first: Int! = 20, $after: String) {
  searchUsers(q: $q, first: $first, after: $after) {
    edges {
      node {
        id
        displayName
        username
        avatarUrl
      }
    }
    pageInfo {
      endCursor
      hasNextPage
    }
  }
}
```

**Variables:** `{ "q": "minh", "first": 20 }`

**Note:** `SearchUsers` is also used by the Invite Search Modal in screen_6 and screen_8 — same endpoint, same query shape.

**Errors:**

| Code | Scenario |
|------|----------|
| `QUERY_TOO_SHORT` | Query is less than 2 characters |

---

### BP. Query: `SearchPets`

```graphql
query SearchPets($q: String!, $first: Int! = 20, $after: String) {
  searchPets(q: $q, first: $first, after: $after) {
    edges {
      cursor
      score         # Float! — relevance score (ADR-0026); node is nullable (null if index stale / entity deleted)
      node {
        id
        name
        species {
          name
          iconEmoji
        }
        breed {
          nameVi
          nameEn
        }
        avatarUrl
        family {
          id
          name
          avatarUrl
        }
      }
    }
    pageInfo {
      endCursor
      hasNextPage
    }
  }
}
```

**Variables:** `{ "q": "british", "first": 20 }`

**Note:** Only pets from public families are returned. Server filters based on family privacy settings.

> **ADR-0026 null-guard:** `SearchFamilies` and `SearchPets` use typed search edges where `node` is **nullable** — client must null-check each edge's `node` before rendering (null means the index is stale or the entity has been deleted since indexing). `SearchUsers` uses a plain `UserConnection` and is not affected.

**Errors:**

| Code | Scenario |
|------|----------|
| `QUERY_TOO_SHORT` | Query is less than 2 characters |

---

## Edge Cases

| Case | Behaviour |
|------|-----------|
| Query matches both name and handle (`@tag` for families / `@username` for users) | Return all matches regardless of which field matched |
| User searches their own @username | They appear in Users results normally |
| Pet has no breed (breed=null) | Still searchable by species only |
| Family is charity | Show CHARITY badge + Donate button in result row |
| User already follows a family in results | Follow button shows "Following" state |
| Pagination | 20 results per tab per page; infinite scroll to load more |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Auth required | Yes — unauth redirected to Login |
| 2 | Min characters | 2 characters |
| 3 | Search scope | Families, Users, Pets only — no post search (caption full-text search too broad to index) |
| 4 | Search criteria | Families: name/@tag; Users: name/@username; Pets: breed/species |
| 5 | Result display | Tabbed — one tab per entity type |
| 6 | Pet privacy | Only pets with `isPublic = true` AND at least 1 visible post returned |
