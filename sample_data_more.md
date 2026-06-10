# Sample Data — More tab (Pet Friendly & Events)

Reference seed data for the **admin-entered** More categories, so backend can define the API and seed a realistic dataset. All records are in **HCMC** (`cityCode: "HCM"`, the only `AVAILABLE` city this phase).

- **Pet Friendly** field model → `screen_21`; query shapes → `PetFriendlyPlaces (CD)` / `PetFriendlyPlace (CE)` (`screen_17` / `screen_21`).
- **Events** field model → `screen_24`; query shapes → `Events (CI)` / `Event (CJ)` (`screen_17` / `screen_24`).

> `distanceKm` is **computed per request** (from the caller's GPS or city centre) — it is **not** seeded. `avgRating` / `reviewCount` (Pet Friendly) and `interestedCount` / `viewerInterested` (Events) are **computed/aggregated**, not seeded (shown here only to illustrate the read shape). City centre used for these samples: HCMC `lat 10.7769, lng 106.7009`.

---

## A. Pet Friendly places

Field model (admin enters): `name`, `category`, `photos[]`, `description?`, `address?`, `city`, `lat`, `lng`, `suitableFor[]`, `tagText?` (≤18), `highlightText?` (≤20), `isPublished`. Computed on read: `avgRating`, `reviewCount`, `cityShortName`, `cityCode`, `country`, `countryCode`, `distanceKm`.

```json
[
  {
    "id": "place_001",
    "name": "Lava Cat Coffee",
    "category": "CAFE",
    "photos": [
      "https://cdn.petapp.com/places/place_001/1.jpg",
      "https://cdn.petapp.com/places/place_001/2.jpg"
    ],
    "description": "Cozy cat café with 12 resident cats and a quiet upstairs reading nook.",
    "address": "12 Nguyễn Huệ, Bến Nghé, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7731,
    "lng": 106.7042,
    "suitableFor": ["CAT"],
    "tagText": "Carriers OK",
    "highlightText": "open until 22:00",
    "isPublished": true,
    "avgRating": 4.8,
    "reviewCount": 126
  },
  {
    "id": "place_002",
    "name": "Ailu Cat Café",
    "category": "CAFE",
    "photos": ["https://cdn.petapp.com/places/place_002/1.jpg"],
    "description": "Resident-cat café; play with 12 cats while you work.",
    "address": "89 Lê Lợi, Bến Thành, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7700,
    "lng": 106.7008,
    "suitableFor": ["CAT"],
    "tagText": "Resident cats",
    "highlightText": "play with 12 cats",
    "isPublished": true,
    "avgRating": 4.7,
    "reviewCount": 98
  },
  {
    "id": "place_003",
    "name": "The Pet Bistro",
    "category": "RESTAURANT",
    "photos": ["https://cdn.petapp.com/places/place_003/1.jpg", "https://cdn.petapp.com/places/place_003/2.jpg"],
    "description": "Dog-friendly bistro with a shaded patio and a dog menu.",
    "address": "45 Thảo Điền, Quận 2 (Thủ Đức)",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.8045,
    "lng": 106.7390,
    "suitableFor": ["DOG"],
    "tagText": "Patio · dogs OK",
    "highlightText": "dog menu",
    "isPublished": true,
    "avgRating": 4.5,
    "reviewCount": 54
  },
  {
    "id": "place_004",
    "name": "Saigon Pet Hotel & Spa",
    "category": "HOTEL",
    "photos": ["https://cdn.petapp.com/places/place_004/1.jpg"],
    "description": "Boarding + grooming for cats and dogs; 24/7 staff.",
    "address": "120 Phan Xích Long, Phú Nhuận",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7990,
    "lng": 106.6850,
    "suitableFor": ["CAT", "DOG"],
    "tagText": "Boarding · grooming",
    "highlightText": "24/7 staff",
    "isPublished": true,
    "avgRating": 4.6,
    "reviewCount": 71
  },
  {
    "id": "place_005",
    "name": "Tao Đàn Dog Park Corner",
    "category": "PARK",
    "photos": ["https://cdn.petapp.com/places/place_005/1.jpg"],
    "description": "Open green area popular for off-leash morning dog walks.",
    "address": "Tao Đàn Park, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7770,
    "lng": 106.6920,
    "suitableFor": ["DOG", "OTHER"],
    "tagText": "Off-leash 5–8am",
    "highlightText": "large open area",
    "isPublished": true,
    "avgRating": 4.4,
    "reviewCount": 33
  },
  {
    "id": "place_006",
    "name": "Hồ Tràm Pet Beach (demo, out of city)",
    "category": "BEACH",
    "photos": ["https://cdn.petapp.com/places/place_006/1.jpg"],
    "description": "Demo record outside HCMC — should NOT appear in HCMC listings.",
    "address": "Hồ Tràm, Bà Rịa–Vũng Tàu",
    "city": "Vũng Tàu",
    "cityShortName": "Vũng Tàu",
    "cityCode": "VT",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.4680,
    "lng": 107.4300,
    "suitableFor": ["DOG"],
    "tagText": "Sand · water",
    "highlightText": "200kđ pet fee",
    "isPublished": true,
    "avgRating": 4.9,
    "reviewCount": 12
  }
]
```

**Notes for BE:**
- `category` enum: `CAFE` / `RESTAURANT` / `HOTEL` / `PARK` / `BEACH` / `OTHER`.
- `suitableFor` enum: `CAT` / `DOG` / `OTHER` (multi).
- `tagText` ≤ 18 chars, `highlightText` ≤ 20 chars (enforce on admin input).
- `place_006` is intentionally **outside HCMC** — a query for `{ cityCode: "HCM" }` must **exclude** it (validates city scoping).

---

## B. Events

Field model (admin enters): `title`, `photos[]`, `description?`, `startAt`, `endAt`, `price?`, `isFree`, `venueName`, `address`, `city`, `lat`, `lng`, `isPublished`. Computed on read: `cityShortName`, `cityCode`, `country`, `countryCode`, `distanceKm`, `interestedCount`, `viewerInterested`.

> Times are ISO 8601 (UTC `Z`). HCMC local = UTC+7, so `02:00Z` = `09:00` local. Pick `startAt`/`endAt` so the dataset spans **today / this weekend / next week** to exercise the time-window filters and the `Starts in / Ends in` countdown. Treat "now" ≈ `2026-06-13` when reading these.

```json
[
  {
    "id": "event_001",
    "title": "Cat Adoption Day",
    "photos": ["https://cdn.petapp.com/events/event_001/1.jpg", "https://cdn.petapp.com/events/event_001/2.jpg"],
    "description": "Ngày hội nhận nuôi mèo do PetApp phối hợp các trạm cứu hộ tổ chức. Mang theo CMND nếu muốn nhận bé về.",
    "startAt": "2026-06-15T02:00:00Z",
    "endAt": "2026-06-15T05:00:00Z",
    "price": "Free",
    "isFree": true,
    "venueName": "Lava Cat Coffee",
    "address": "12 Nguyễn Huệ, Bến Nghé, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7731,
    "lng": 106.7042,
    "isPublished": true,
    "interestedCount": 48,
    "viewerInterested": false
  },
  {
    "id": "event_002",
    "title": "Dog Run Meetup",
    "photos": ["https://cdn.petapp.com/events/event_002/1.jpg"],
    "description": "Buổi chạy bộ cùng các bé cún tại công viên. Mọi giống chó thân thiện đều chào đón.",
    "startAt": "2026-06-18T10:30:00Z",
    "endAt": "2026-06-18T12:00:00Z",
    "price": "Free",
    "isFree": true,
    "venueName": "Tao Đàn Park",
    "address": "Tao Đàn Park, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7770,
    "lng": 106.6920,
    "isPublished": true,
    "interestedCount": 23,
    "viewerInterested": false
  },
  {
    "id": "event_003",
    "title": "Pet Health Workshop",
    "photos": ["https://cdn.petapp.com/events/event_003/1.jpg"],
    "description": "Bác sĩ thú y chia sẻ cách chăm sóc sức khỏe cơ bản cho chó mèo. Có Q&A cuối buổi.",
    "startAt": "2026-06-22T07:00:00Z",
    "endAt": "2026-06-22T09:00:00Z",
    "price": "200kđ",
    "isFree": false,
    "venueName": "PetCare Clinic Q1",
    "address": "67 Hai Bà Trưng, Bến Nghé, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7820,
    "lng": 106.7010,
    "isPublished": true,
    "interestedCount": 12,
    "viewerInterested": true
  },
  {
    "id": "event_004",
    "title": "Cat Lovers Saturday Market",
    "photos": ["https://cdn.petapp.com/events/event_004/1.jpg", "https://cdn.petapp.com/events/event_004/2.jpg"],
    "description": "Phiên chợ cuối tuần: phụ kiện, đồ ăn, và khu giao lưu cho hội mê mèo.",
    "startAt": "2026-06-13T03:00:00Z",
    "endAt": "2026-06-13T10:00:00Z",
    "price": "Free",
    "isFree": true,
    "venueName": "Saigon Pet Plaza",
    "address": "Tầng 2, 8 Lê Duẩn, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7815,
    "lng": 106.7000,
    "isPublished": true,
    "interestedCount": 65,
    "viewerInterested": false
  },
  {
    "id": "event_005",
    "title": "Charity Pet Photo Day",
    "photos": ["https://cdn.petapp.com/events/event_005/1.jpg"],
    "description": "Chụp ảnh cho thú cưng gây quỹ cứu hộ. Toàn bộ phí ủng hộ trạm cứu hộ địa phương.",
    "startAt": "2026-06-27T06:00:00Z",
    "endAt": "2026-06-27T11:00:00Z",
    "price": "150kđ",
    "isFree": false,
    "venueName": "The Pet Bistro",
    "address": "45 Thảo Điền, Quận 2 (Thủ Đức)",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.8045,
    "lng": 106.7390,
    "isPublished": true,
    "interestedCount": 9,
    "viewerInterested": false
  },
  {
    "id": "event_006",
    "title": "Past Event — Puppy Training 101 (ended)",
    "photos": ["https://cdn.petapp.com/events/event_006/1.jpg"],
    "description": "Demo record in the past — should NOT appear in any upcoming listing.",
    "startAt": "2026-06-06T07:00:00Z",
    "endAt": "2026-06-06T09:00:00Z",
    "price": "Free",
    "isFree": true,
    "venueName": "Tao Đàn Park",
    "address": "Tao Đàn Park, Quận 1",
    "city": "Ho Chi Minh City",
    "cityShortName": "HCMC",
    "cityCode": "HCM",
    "country": "Việt Nam",
    "countryCode": "VN",
    "lat": 10.7770,
    "lng": 106.6920,
    "isPublished": true,
    "interestedCount": 40,
    "viewerInterested": false
  }
]
```

**Notes for BE:**
- Return only **upcoming/ongoing** events (`endAt >= now`), sorted `startAt` asc → `event_006` (ended) must be **excluded** from `Events (CI)` listings.
- `isFree = true` → render `"Free"` and match the **Free** filter chip; `isFree = false` → show `price` text.
- `timeWindow` filters: `THIS_WEEK` / `THIS_WEEKEND` computed from `startAt` relative to the caller's "now".
- `interestedCount` / `viewerInterested` are **derived** (from the interest records), not admin-entered — included here only to show the read shape; do not seed them as admin fields.
- Use these to exercise the row countdown: relative to now ≈ `2026-06-13`, `event_004` is *Happening now / today*, `event_001` *Starts in ~2d*, `event_002` *~5d*, etc.

---

## Coverage checklist (what this dataset exercises)

| Scenario | Covered by |
|----------|-----------|
| Multiple Pet Friendly categories | CAFE / RESTAURANT / HOTEL / PARK / BEACH |
| `suitableFor` single & multi | place_001 (CAT) … place_004 (CAT+DOG), place_005 (DOG+OTHER) |
| City scoping (exclude out-of-city) | place_006 (Vũng Tàu) excluded from HCM |
| Free vs paid events | event_001/002/004 Free; event_003/005 paid |
| Time windows (today / weekend / next week) | event_004 (today), event_001/002, event_003/005 |
| Ended event excluded | event_006 |
| Countdown states | derived from `startAt`/`endAt` above |
