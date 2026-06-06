# Screen 4: Register / Login

## Overview

Entry point for unauthenticated users. Supports two registration/login methods:
1. **Phone number → OTP**
2. **Google / Apple ID (OAuth)**

After first-time registration via any method, user is prompted to complete their profile (name, tag, avatar) before proceeding to the app.

---

## UI Layout

```
[Logo / App name]

[Phone number input]  ← primary method
[Continue button]

─── or ───

[Sign in with Google]
[Sign in with Apple]

[Terms of Service / Privacy Policy links]
```

---

## Flows

### Flow A — Phone + OTP

```
1. User enters phone number → tap Continue
     └─> POST /auth/phone/request-otp  { phone: "+84901234567" }
           ├─ 400 INVALID_PHONE → show inline error
           └─ 200 → navigate to OTP screen

2. OTP screen: 6-digit input + countdown timer (60s resend)
     └─> POST /auth/phone/verify-otp  { phone, otp }
           ├─ 400 INVALID_OTP → show error, allow retry
           ├─ 429 TOO_MANY_ATTEMPTS → show "Too many attempts, try again later"
           └─ 200 → { access_token, refresh_token, is_new_user }
                 ├─ is_new_user=true  → navigate to Profile Setup screen
                 └─ is_new_user=false → navigate to Explore (home)

3. Resend OTP: available after 60s countdown
     └─> POST /auth/phone/request-otp (same as step 1)
```

### Flow B — Google / Apple OAuth

```
1. User taps "Sign in with Google" / "Sign in with Apple"
     └─> Native OAuth flow (handled by SDK)
           └─> POST /auth/oauth  { provider: "google"|"apple", id_token: "..." }
                 └─ 200 → { access_token, refresh_token, is_new_user }
                       ├─ is_new_user=true  → navigate to Profile Setup screen
                       └─ is_new_user=false → navigate to Explore (home)
```

---

## Profile Setup Screen *(first-time only, after registration)*

Shown once after a new account is created via any method.

```
[Avatar picker — optional, can skip]
[Name input — required]
[Tag input — required, unique, format: letters/numbers/underscore only, no spaces]
[Continue button]
```

| Field | Rules |
|-------|-------|
| `avatar` | Optional; user can upload or skip (default avatar used) |
| `name` | Required; max 50 chars |
| `tag` | Required; unique across all users; only set once at creation — **cannot be changed after this step** |

**Tag validation:**
- Real-time uniqueness check as user types (debounced 500ms)
- `GET /users/check-tag?tag=minhdang` → `{ available: true|false }`
- Show green checkmark if available, red X if taken

**Submit:**
- `PATCH /users/me` `{ name, tag, avatar_url }` (avatar uploaded separately via media endpoint first)
- On success → navigate to Explore

---

## API Endpoints Required

### AA. `POST /auth/phone/request-otp`
**Body:** `{ "phone": "+84901234567" }`  
**Response `200 OK`:** `{ "expires_in": 60 }`

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_PHONE` | Phone format invalid |
| `429` | `RATE_LIMITED` | Too many OTP requests for this number |

---

### AB. `POST /auth/phone/verify-otp`
**Body:** `{ "phone": "+84901234567", "otp": "123456" }`  
**Response `200 OK`:**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "is_new_user": true
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_OTP` | OTP incorrect |
| `400` | `OTP_EXPIRED` | OTP has expired |
| `429` | `TOO_MANY_ATTEMPTS` | Too many wrong attempts |

---

### AC. `POST /auth/oauth`
**Body:** `{ "provider": "google", "id_token": "..." }`  
**Response `200 OK`:** Same shape as `verify-otp` response.

---

### AD. `GET /users/check-tag`
**Query:** `?tag=minhdang`  
**Auth:** Not required  
**Response `200 OK`:** `{ "available": true }`

---

### AE. `PATCH /users/me`
Update user profile (used for profile setup and future edits).  
**Auth:** Required  
**Body:** `{ "name": "Minh Dang", "tag": "minhdang", "avatar_url": "https://..." }`  
**Response `200 OK`:** Updated user object.

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `409` | `TAG_TAKEN` | Tag already in use by another user |
| `400` | `TAG_ALREADY_SET` | User attempts to change tag after it was already set |

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| User closes app mid-OTP | Phone + OTP screen state is lost; user must restart from phone entry |
| OTP expired before entry | Show "Code expired" with Resend button immediately active |
| Tag already set, user tries to change | `400 TAG_ALREADY_SET`; hide tag field on Edit Profile for existing users |
| OAuth account already registered | Treated as login (`is_new_user=false`); skip profile setup |
| User skips avatar on profile setup | Default avatar (initials or placeholder) used until manually updated |
