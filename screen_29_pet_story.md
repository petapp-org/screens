# Screen 29: Pet Story

## Overview

A **chronological photo/video timeline of a single pet** — every post that tags this pet, grouped by month and day, with a one-tap **Play** slideshow. Think "Bụi's Story": a living scrapbook built automatically from the family's posts.

**Member-only.** The Story button and this screen are visible **only to members of the pet's family** (owner + parents). It is **not** public, even for a public pet. Accessed from:
- **My Pets** (`screen_8`) → pet row → **Story** button,
- **Pet Detail** (`screen_9`) → **Story** button.

> The Story button on the public **Family Posts** screen (`screen_3`) stays **hidden** — that screen's viewer is always a non-member (see `screen_3`).

Requires login (member context).

---

## Concept & Data Rules

- The story is assembled from **posts that tag this pet** (a `media_tag` of type `pet` matching this pet — not "random" detections). Posts/media that are soft-deleted are excluded.
- **One square = one post.** The square's cover image is **this pet's first media in that post** (decision). So even on a multi-pet post, the story shows *this* pet, never the other pets' media.
- **Play** plays **every one of this pet's media items** across all those posts (not just the covers), oldest → newest.

> **Multi-pet post — the two rules:**
> 1. In *this* pet's story, the square/cover uses **this pet's first media** in the post (if a post has `[dog1, dog2, cat1, cat2]`, the cat's story shows `cat1`).
> 2. **Opening the post from here reorders its media** so this pet's media come first — the Post Detail is opened with a `focusPetId` (see `screen_2`).

---

## UI Layout

```
[Header]
  ←   (avatar) Bụi's Story                         [ ▶ Play ]
      26 photos · 2 videos · 2022 – now

[MAR 2022 ▾]                                        ← month group header (collapse/expand)
  ● Tue, Mar 8 · 3 posts                            ← row = one day (timeline dot)
     [□][□][□]                                       ← one square per post that day
  ● Sun, Mar 20 · 2 posts
     [□][□]
  ● Sun, Mar 27 · 1 post
     [    □ big    ]

[APR 2022 ▾]
  ● ...

  (Empty: "No story yet — posts that tag Bụi will show up here.")

[ ↑ Newest ]   (floating pill, bottom-left)         ← jump to newest / oldest
```

---

## Components

### 1. Header

| Element | Display |
|---------|---------|
| Back | Returns to the entry screen (My Pets / Pet Detail) |
| Pet avatar + `{name}'s Story` | e.g. `"Bụi's Story"` |
| Sub-label | `"{photoCount} photos · {videoCount} videos · {yearRange}"` — counts split by media type; `yearRange` = `"{firstYear} – now"` (or `"{firstYear} – {lastYear}"`) |
| **[▶ Play]** | Top-right — starts the slideshow (Section 4) |

> The sub-label counts **media items** split by type (e.g. `"26 photos · 2 videos"`). A type with **0** is **omitted** (e.g. `"26 photos · 2022 – now"` when there are no videos). The total (`photoCount + videoCount`) = what **Play** iterates. (The grid below shows one **cover per post**, which may be fewer.)

---

### 2. Month Groups (collapsible)

- Posts are bucketed by **month**, newest months **below** (the whole screen is **oldest-first**, top = oldest).
- Each month header (`"MAR 2022"`) has a chevron: **tap to collapse/expand** all of that month's days. `▾` = expanded, `›` = collapsed.
- Default: expanded.

---

### 3. Day Rows + Post Squares

Within a month, each **day** that has ≥ 1 qualifying post is a **row** on a vertical timeline (a dot on the left rail):

```
● {Weekday, Mon D} · {N} posts
   [square][square]…
```

- **Row label:** `"Tue, Mar 8 · {N} posts"` — `N` = number of this pet's posts that day.
- **Squares — one per post that day** (cover = this pet's first media in the post):

| Posts that day | Layout |
|----------------|--------|
| 1 | one **large** square (full-width) |
| 2 | two medium squares in a row |
| 3 | three squares in a row |
| > 3 | horizontal-scroll row of squares |

- A square whose cover is a **video** shows a small **▶ badge**.
- **Tap a square** → open **Post Detail** (`screen_2`) for that post, passing `focusPetId = this pet` so its media are reordered to the front.

---

### 4. Play (slideshow)

A fullscreen, auto-advancing slideshow of **this pet's media**, **oldest → newest**.

| Media | Duration | Notes |
|-------|----------|-------|
| Photo | **3 seconds** | subtle Ken-Burns zoom for life |
| Video | **first 5 seconds** | play the first 5s of the clip; if shorter, play the whole clip |

**Controls** (IG-Stories style):
- Segmented **progress bar** at the top (one segment per media item).
- **Tap right / left** → next / previous; **hold** → pause; **swipe down or ×** → exit.
- On reaching the end → show a short "end card" (e.g. *"That's Bụi's story so far 🐾"*) then exit.
- *(Optional, later: background music track.)*

---

### 5. Newest / Oldest Jump (floating pill)

- The screen opens **oldest-first** (top = oldest). A floating pill, bottom-left, lets the user jump:
  - When scrolled near the **oldest** end → label **"↑ Newest"** → tap scrolls to the **bottom** (newest).
  - When near the **newest** end → label **"↓ Oldest"** → tap scrolls to the **top** (oldest).
- (Position-aware single control — no sort change; order is always oldest-first.)

---

## API Endpoints Required

> All calls go to `POST /graphql`. **Auth:** Required (family member).

---

### CT. Query: `PetStory`

Fetch the full story for a pet — months → days → posts, each post carrying **this pet's** ordered media (cover = index 0; the flat sequence powers Play).

**Operation:**
```graphql
query PetStory($petId: ID!) {
  petStory(petId: $petId) {
    pet { id name avatarUrl }
    photoCount             # this pet's IMAGE media count → header "{n} photos"
    videoCount             # this pet's VIDEO media count → header "{n} videos" (omit if 0)
    firstYear
    lastYearLabel          # "now" if there's a post this year, else the year
    months {
      label                # "MAR 2022"
      year
      month                # 3
      days {
        date               # "2022-03-08"
        label              # "Tue, Mar 8"
        posts {
          postId
          postedAt
          petMedia {       # THIS pet's media in the post, in post order; [0] = cover
            url
            type           # IMAGE | VIDEO
            thumbnailUrl   # for video cover
          }
        }
      }
    }
  }
}
```

**Notes:**
- `months` ordered **oldest-first**; `days` oldest-first within a month; `posts` oldest-first within a day.
- `posts[].petMedia` is **only this pet's** media in that post (filtered by `media_tag` = this pet), preserving the post's media order; `petMedia[0]` is the square cover.
- **Square cover** = `petMedia[0]` (use `thumbnailUrl` + ▶ badge when `type = VIDEO`).
- **Play playlist** = concatenate every `petMedia` across all posts in chronological order; length = `photoCount + videoCount`.
- Only posts with ≥ 1 media tagged to this pet are returned; soft-deleted posts/media excluded.

**Response `200 OK` (excerpt):**
```json
{
  "data": {
    "petStory": {
      "pet": { "id": "pet_111", "name": "Bụi", "avatarUrl": "https://cdn.petapp.com/pets/pet_111/avatar.jpg" },
      "photoCount": 26,
      "videoCount": 2,
      "firstYear": 2022,
      "lastYearLabel": "now",
      "months": [
        {
          "label": "MAR 2022",
          "year": 2022,
          "month": 3,
          "days": [
            {
              "date": "2022-03-08",
              "label": "Tue, Mar 8",
              "posts": [
                { "postId": "post_a1", "postedAt": "2022-03-08T09:00:00Z",
                  "petMedia": [ { "url": "…/a1_1.jpg", "type": "IMAGE", "thumbnailUrl": null } ] },
                { "postId": "post_a2", "postedAt": "2022-03-08T15:00:00Z",
                  "petMedia": [ { "url": "…/a2_1.jpg", "type": "IMAGE", "thumbnailUrl": null } ] },
                { "postId": "post_a3", "postedAt": "2022-03-08T18:00:00Z",
                  "petMedia": [ { "url": "…/a3_v.mp4", "type": "VIDEO", "thumbnailUrl": "…/a3_thumb.jpg" } ] }
              ]
            }
          ]
        }
      ]
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `401` | `UNAUTHENTICATED` | Not logged in |
| `403` | `NOT_FAMILY_MEMBER` | Caller is not a member of the pet's family (Story is member-only) |
| `404` | `PET_NOT_FOUND` | Pet does not exist or is soft-deleted |

> **Post reorder:** opening a square uses the existing Post query (`screen_2` → `Post (M)`) with a new optional `focusPetId` — see `screen_2`. No new endpoint for that.

---

## User Flow Diagrams

### Open story

```
My Pets / Pet Detail → [Story]   (member only)
  └─> PetStory (CT) { petId }
        ├─ has posts → render months → days → post squares (oldest-first)
        └─ none → "No story yet" empty state
```

### Open a post (multi-pet reorder)

```
Tap a post square
  └─> Post Detail (screen_2) { postId, focusPetId: thisPet }
        └─> media reordered so this pet's media are first
```

### Play

```
Tap [▶ Play]
  └─> fullscreen slideshow of this pet's media, oldest → newest
        ├─ photo → 3s (Ken Burns)
        ├─ video → first 5s (or full if shorter)
        └─ progress segments · tap next/prev · hold pause · swipe-down/× exit → end card
```

### Navigate

```
Tap a month header        → collapse / expand that month's days
Tap "↑ Newest" pill        → scroll to bottom (newest); label flips to "↓ Oldest"
```

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| Not a family member | Story button hidden; direct nav → `NOT_FAMILY_MEMBER` |
| Pet has no qualifying posts | "No story yet — posts that tag {pet} will show up here." |
| Multi-pet post | Square/cover = this pet's first media; Play uses only this pet's media; post opens reordered |
| Post has several of this pet's media | Still **one** square (cover = first); all of them play in Play |
| Video cover | Show ▶ badge; Play trims to first 5s |
| Day with > 3 posts | Horizontal-scroll square row |
| Month collapsed | Days hidden; header shows `›` |
| Soft-deleted post / media | Excluded from the story and Play |
| Private / followers-only post | Included — viewer is always a family member (sees all the family's posts) |
| Pet soft-deleted | Story not reachable (`PET_NOT_FOUND`) |
| Tap square | Post Detail with `focusPetId` (reordered media) |

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Visibility | **Member-only** (owner + parents); hidden from non-members and on the public Family Posts screen |
| 2 | Grouping | Month groups (collapsible) → day rows → **one square per post** |
| 3 | Square cover | **This pet's first media** in the post (multi-pet safe) |
| 4 | Square layout per day | 1 → large · 2 → two · 3 → three · > 3 → horizontal scroll |
| 5 | Tap square | Open Post Detail with `focusPetId` → media reordered (this pet first) |
| 6 | Play | This pet's media oldest→newest; **photo 3s** (Ken Burns), **video first 5s**; IG-style controls |
| 7 | Order + jump | Always **oldest-first**; floating position-aware pill jumps to Newest/Oldest |
| 8 | Header count | Split by type: `"{photoCount} photos · {videoCount} videos · {yearRange}"` (omit a 0 type); total = Play length; grid shows one cover per post |
| 9 | What counts | Posts tagging this pet (named, not random); soft-deleted excluded |
| 10 | Entry points | My Pets pet row + Pet Detail; **not** Family Posts (non-member view) |

---

## Open Items (next steps)

- **Background music** for Play (Apple-Memories style) — optional later.
- **Share / export** a story or a Play video — not this phase (member-only for now).
- **Per-year jump** / scrubber if stories get very long — month collapse covers it for now.
