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

All operations use `POST /graphql`. Auth where required via `Authorization: Bearer <token>` header.

**Shared types:**
```graphql
type AuthResult {
  accessToken: String!
  refreshToken: String!
  user: User!
  isNewUser: Boolean!
  requiresProfileSetup: Boolean!
}

type User {
  id: ID!
  displayName: String
  tag: String
  avatarUrl: String
}
```

---

### AA. Mutation: `RequestOtp`

Request an OTP be sent to the given phone number.
**Auth:** Not required

**Operation:**
```graphql
mutation RequestOtp($input: RequestOtpInput!) {
  requestOtp(input: $input) {
    expiresIn
  }
}
```

**Variables:**
```json
{ "input": { "phone": "+84901234567" } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "requestOtp": {
      "expiresIn": 60
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `INVALID_PHONE` | Phone format invalid |
| `RATE_LIMITED` | Too many OTP requests for this number |

---

### AB. Mutation: `VerifyOtp`

Verify the OTP and authenticate the user. Returns tokens and new-user flag.
**Auth:** Not required

**Operation:**
```graphql
mutation VerifyOtp($input: VerifyOtpInput!) {
  verifyOtp(input: $input) {
    accessToken
    refreshToken
    isNewUser
    requiresProfileSetup
    user {
      id
      displayName
      tag
      avatarUrl
    }
  }
}
```

**Variables:**
```json
{ "input": { "phone": "+84901234567", "otp": "123456" } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "verifyOtp": {
      "accessToken": "eyJ...",
      "refreshToken": "eyJ...",
      "isNewUser": true,
      "requiresProfileSetup": true,
      "user": {
        "id": "u_abc123",
        "displayName": null,
        "tag": null,
        "avatarUrl": null
      }
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `INVALID_OTP` | OTP incorrect |
| `OTP_EXPIRED` | OTP has expired |
| `TOO_MANY_ATTEMPTS` | Too many wrong attempts |

---

### AC. Mutation: `AuthWithGoogle`

Authenticate via Google OAuth id_token. Returns same `AuthResult` shape.
**Auth:** Not required

**Operation:**
```graphql
mutation AuthWithGoogle($input: OAuthInput!) {
  authWithGoogle(input: $input) {
    accessToken
    refreshToken
    isNewUser
    requiresProfileSetup
    user {
      id
      displayName
      tag
      avatarUrl
    }
  }
}
```

**Variables:**
```json
{ "input": { "idToken": "eyJ..." } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "authWithGoogle": {
      "accessToken": "eyJ...",
      "refreshToken": "eyJ...",
      "isNewUser": false,
      "requiresProfileSetup": false,
      "user": {
        "id": "u_abc123",
        "displayName": "Minh Dang",
        "tag": "minhdang",
        "avatarUrl": "https://..."
      }
    }
  }
}
```

---

### AD. Mutation: `AuthWithApple`

Authenticate via Apple OAuth id_token. Returns same `AuthResult` shape.
**Auth:** Not required

**Operation:**
```graphql
mutation AuthWithApple($input: OAuthInput!) {
  authWithApple(input: $input) {
    accessToken
    refreshToken
    isNewUser
    requiresProfileSetup
    user {
      id
      displayName
      tag
      avatarUrl
    }
  }
}
```

**Variables:**
```json
{ "input": { "idToken": "eyJ..." } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "authWithApple": {
      "accessToken": "eyJ...",
      "refreshToken": "eyJ...",
      "isNewUser": true,
      "requiresProfileSetup": true,
      "user": {
        "id": "u_abc123",
        "displayName": null,
        "tag": null,
        "avatarUrl": null
      }
    }
  }
}
```

---

### AE. Mutation: `SetupProfile`

Update user profile (used for profile setup and future edits).
**Auth:** Required

**Operation:**
```graphql
mutation SetupProfile($input: SetupProfileInput!) {
  setupProfile(input: $input) {
    id
    displayName
    tag
    avatarUrl
  }
}
```

**Variables:**
```json
{ "input": { "displayName": "Minh Dang", "tag": "minhdang", "avatarUrl": "https://..." } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "setupProfile": {
      "id": "u_abc123",
      "displayName": "Minh Dang",
      "tag": "minhdang",
      "avatarUrl": "https://..."
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `TAG_TAKEN` | Tag already in use by another user |
| `TAG_ALREADY_SET` | User attempts to change tag after it was already set |

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| User closes app mid-OTP | Phone + OTP screen state is lost; user must restart from phone entry |
| OTP expired before entry | Show "Code expired" with Resend button immediately active |
| Tag already set, user tries to change | `400 TAG_ALREADY_SET`; hide tag field on Edit Profile for existing users |
| OAuth account already registered | Treated as login (`is_new_user=false`); skip profile setup |
| User skips avatar on profile setup | Default avatar (initials or placeholder) used until manually updated |
