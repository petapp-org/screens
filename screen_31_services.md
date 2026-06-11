# Screen 31: Services

## Overview

The **"Services" tab** — fourth tab in the bottom navigation. A location-aware hub for pet services (grooming, vet, boarding, etc.). **This phase ships a placeholder only** — **"Coming Soon"** in every city.

**Identical to Shops (`screen_30`)** in every respect — same header, location bar, Coming-Soon body, behaviour, API, and auth. The only differences:

| Aspect | Shops (`screen_30`) | Services (this screen) |
|--------|--------------------|------------------------|
| Header title | `"Shops"` | `"Services"` |
| Active bottom-nav tab | Shops | **Services** |
| Tab icon (bottom nav) | shopping-bag | stethoscope |

Everything else — the canonical root-tab header (`screen_8`), the location bar + Choose Your City + `Cities (CA)` (`screen_17`), the Coming-Soon body with **[Change Location]**, the login requirement, and the city-independent Coming-Soon state — is exactly as documented in **`screen_30`**.

---

## UI Layout

```
[Header]
  Title: "Services"
  Right actions: [Search icon] | [Messages icon ✉] | [Notifications icon 🔔] | [Profile avatar]

[Location bar]
  [📍 icon]  Hà Nội, VN                                 [Change]

[Coming Soon body]
            ( 📍 pin glyph in a soft circle )
                     Coming Soon!
        We are currently not available in this region.
                 [ Change Location ]

[Bottom Navigation]
  My Pets | Explore | Shops | Services (active) | More
```

---

## Components / API / Flows / Edge Cases

See **`screen_30` (Shops)** — identical. (Reuses `Cities (CA)` for the location picker; no Services-listing query this phase.)

---

## Decisions Log

| # | Question | Decision |
|---|----------|----------|
| 1 | Relationship to Shops | **Identical to `screen_30`**, differing only in title (`Services`), active tab, and tab icon |
| 2 | Scope this phase | Placeholder only — Coming Soon in every city |

---

## Open Items (next steps)

- **Services feature** (grooming / vet / boarding listings, per-city availability) — future phase; this screen is the placeholder until then.
