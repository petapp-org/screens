# PetApp screen specs — guide for Claude Code / AI agents

This repo holds the **client-facing screen specifications** for the PetApp mobile/web
clients: one `screen_*.md` per screen, each documenting the GraphQL queries/mutations
that screen uses, with example responses and field docs.

These specs are a **consumer view of the API**, not the source of truth. The canonical
GraphQL schema lives in the backend repo at `petapp-be/contract/schema.graphql` (the file
client codegen is generated from). When a spec and the contract disagree, **the contract
is right and the spec is stale** — fix the spec to match the contract, never the reverse.
(The reconcile effort that aligns specs ↔ contract is tracked in `petapp-be#877`.)

---

## GraphQL naming conventions (the contract enforces these 100%)

- **field / argument** → `camelCase` (`familyType`, `firstName`, `scheduledAt`)
- **type / interface / enum / scalar / input** → `PascalCase` (`Family`, `PostConnection`)
- **enum value** → `SCREAMING_SNAKE` (`CHARITY`, `PENDING`, `LATEST`)

Specs are often written with `snake_case` field docs (`is_own`, `follower_count`) or ad-hoc
names — those are drift. Match the exact contract spelling, because the client codegen keys
off it. If unsure of the canonical name, grep the contract:

```sh
grep -nE "<fieldName>|type <TypeName>" ../petapp-be/contract/schema.graphql
```

---

## Editing a field: update ALL representations

A single field can appear in up to **four** places within one screen spec. When you rename
or reshape a field, update every one of them — a partial edit leaves the spec internally
inconsistent (worse than not touching it):

1. **GraphQL selection set** — the `query`/`mutation` code block
2. **JSON response example** — the `"data": { ... }` block
3. **Field-doc table** — the `| field | description |` rows
4. **Prose / ASCII layout** — sentences and UI mockups that reference the field by name
   (e.g. ``shown only when `familyType = charity` ``, or `@username` in a header mockup)

Prose and ASCII-diagram references are the easiest to miss and the most common source of a
half-renamed PR. Grep for them explicitly.

---

## Completeness: grep all variants across ALL screens

Before claiming a rename is done, sweep **every** `screen_*.md`, not a subset, and grep
**every spelling variant**:

```sh
for tok in fieldName field_name fieldVariant; do echo "-- $tok"; grep -rln "$tok" screen_*.md; done
```

**The same token can name different fields on different entities** — read the context of
each hit, change only the right entity, and report ambiguous cases. Known traps:

- `tag` is **`User.tag` → renamed to `username`**, but **`Family.tag` is NOT renamed** —
  Family keeps `tag` / `@tag`. A blanket replace is wrong.
- `type` as a field appears on Family (`familyType`), Media (`Media.type`), and MediaTag
  (`MediaTag.type`) — only Family's was renamed.
- `isActive` exists on both User (canonical, kept) and Family (renamed to `isPrimary`).

---

## Preserve Vietnamese data values

These specs are English documentation, but the **example data is Vietnamese** and must be
kept verbatim: names like `Bụi`, `Nhà Mèo`, `Nhà Test`, places like `Hồ Chí Minh` /
`Việt Nam`, handles like `minhdang`, and emoji in example post/comment bodies
(`Cute quá trời 😍`). Only rename **field keys/names** — never translate or alter a **value**.
Example: renaming `tag` → `username` changes the key but keeps `"username": "minhdang"`.

---

## Pagination — Relay Connection is canonical (ADR-0023)

Every list field in the contract uses the Relay Connection shape. Specs must match it:

```graphql
field(first: Int! = 20, after: String): XConnection!

type XConnection { edges: [XEdge!]!  pageInfo: XPageInfo! }
type XEdge       { cursor: String!   node: X! }
type XPageInfo   { hasNextPage: Boolean!  endCursor: String }
```

- **Args** are `first` / `after`. **Not** `limit` / `cursor` / `offset` / `page` / `pageSize`.
- **PageInfo is per-domain** (`PostPageInfo`, `FamilyPageInfo`) — there is no shared `PageInfo`.
- **No old envelopes.** Shapes like `{ items: [...], nextCursor, hasMore, totalCount }` or
  `{ posts: [...], nextCursor }` are stale drift that was already migrated away. A top-level
  count travels as a sibling field on the parent (e.g. `post.commentsCount`), not inside the
  connection.
- You may still see `limit` / `cursor` args in the contract marked
  `@deprecated("Renamed to first/after (ADR-0023); removed in R+1.")` — those are transitional
  aliases. Treat `first`/`after` as canonical; any spec still using `limit`/`cursor` is stale.
- **Search has a typed-node variant (ADR-0026):** a search edge carries `score: Float!` and a
  `node` that is the **typed entity** (`Pet` / `Family` / `Post`), and `node` is **nullable**
  (null when the index is stale / the entity is gone). Exceptions that still use opaque
  `*SearchHit`: `searchLostPets` / `searchRescueCases` / `searchPlaces` (domains not yet built).
- **New list fields must be born Relay** — when a not-yet-shipped feature (chat, places, events,
  rescue, etc.) gets specified, write its pagination as a Connection from the start; do not
  reintroduce `limit`/`cursor` envelopes.

---

## Branch & PR workflow

- Branch off `main`: `git fetch origin main && git checkout -b spec/<domain>-<change> origin/main`.
- One logical change per PR; reference the backend issue/PR in the body
  (e.g. `petapp-be#877`).
- `gh pr create --base main`. **Do not merge without explicit human confirmation.**
- `gh pr edit` may fail with a `projectCards` GraphQL error on this org; edit a PR body via
  `gh api -X PATCH repos/petapp-org/screens/pulls/<n> -f body="$(cat body.md)"`.
