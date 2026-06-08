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
     └─> RequestOtp mutation (AA) { phone: "+84901234567" }
           ├─ INVALID_PHONE → show inline error
           └─ 200 → navigate to OTP screen

2. OTP screen: 6-digit input + countdown timer (60s resend)
     └─> VerifyOtp mutation (AB) { phone, otp }
           ├─ INVALID_OTP → show error, allow retry
           ├─ TOO_MANY_ATTEMPTS → show "Too many attempts, try again later"
           └─ 200 → { accessToken, refreshToken, isNewUser, requiresProfileSetup }
                 ├─ requiresProfileSetup=true  → navigate to Profile Setup screen
                 └─ requiresProfileSetup=false → navigate to post-login target
                                                  (the return target, else Explore)

3. Resend OTP: available after 60s countdown
     └─> RequestOtp mutation (AA) (same as step 1)
```

### Flow B — Google / Apple OAuth

```
1. User taps "Sign in with Google" / "Sign in with Apple"
     └─> Native OAuth flow (handled by SDK)
           └─> AuthWithGoogle mutation (AC) / AuthWithApple mutation (AD) { idToken: "..." }
                 └─ 200 → { accessToken, refreshToken, isNewUser, requiresProfileSetup }
                       ├─ requiresProfileSetup=true  → navigate to Profile Setup screen
                       └─ requiresProfileSetup=false → navigate to post-login target
                                                        (the return target, else Explore)
```

---

> **`isNewUser` vs `requiresProfileSetup`:** routing decisions use **`requiresProfileSetup`**, not `isNewUser`. `isNewUser` is just analytics (was the account created on this call). `requiresProfileSetup` is `true` until the user has a name **and** tag — so a user who registered but abandoned setup will have `isNewUser=false` yet still be routed back into Profile Setup on their next login.

> **Return target (post-login redirect):** When the user reaches this screen because an auth-required action redirected them (e.g. tapped Follow / Message / My Pets), that origin is remembered as a **return target**. After successful auth — and after Profile Setup if `requiresProfileSetup=true` — the app navigates **back to the return target**. If there is no return target (app opened cold on Login), it lands on **Explore**.

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
- Uses `CheckUserTag` query (see endpoint AF below)
- Show green checkmark if available, red X if taken
- Show a persistent helper text below the field: *"⚠️ Your tag cannot be changed after you create your account."*

**Submit:**
- If an avatar was picked: upload it first via `RequestMediaUpload (BV)` `{ purpose: "USER_AVATAR" }` → use the returned `publicUrl` (no AI scan — avatars are never scanned)
- `SetupProfile mutation (AE)` `{ name, tag, avatar_url }`
- On success → navigate to the **post-login target** (return target if redirected here, else Explore)

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

### AF. Query: `CheckUserTag`

Check tag availability during Profile Setup.  
**Auth:** Not required

**Operation:**
```graphql
query CheckUserTag($tag: String!) {
  checkUserTag(tag: $tag) {
    available
  }
}
```

**Variables:**
```json
{
  "tag": "minhdang"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "checkUserTag": {
      "available": true
    }
  }
}
```

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_TAG_FORMAT` | Tag contains invalid characters |

---

### BV. Mutation: `RequestMediaUpload`

Get a pre-signed URL to upload an image/video. **Shared by all uploads** — user avatar, family avatar, pet avatar, and post media. **No AI scan** is performed here; scanning post media for pet detection is a separate, post-media-only step (`ScanMedia (AT)`, screen_7). Avatars are never scanned.

**Auth:** Required

**Operation:**
```graphql
mutation RequestMediaUpload($input: MediaUploadInput!) {
  requestMediaUpload(input: $input) {
    uploadUrl   # pre-signed PUT URL — client uploads the raw bytes here
    publicUrl   # final hosted URL to store on the profile/family/pet/post
    expiresIn
  }
}
```

**Variables:**
```json
{ "input": { "contentType": "image/jpeg", "purpose": "USER_AVATAR" } }
```

`purpose` enum: `USER_AVATAR` | `FAMILY_AVATAR` | `PET_AVATAR` | `POST_MEDIA`

**Client flow:** call this → `PUT` the file bytes to `uploadUrl` → use `publicUrl` in the next mutation (`SetupProfile`, `UpdateMe`, family/pet update, or `CreatePost`). For **post media only**, pass `publicUrl` to `ScanMedia (AT)` to detect/match a pet.

**Response `200 OK`:**
```json
{
  "data": {
    "requestMediaUpload": {
      "uploadUrl": "https://storage.petapp.com/upload/abc123?sig=...",
      "publicUrl": "https://cdn.petapp.com/media/abc123.jpg",
      "expiresIn": 300
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `UNAUTHENTICATED` | Caller is not logged in |
| `UNSUPPORTED_MEDIA_TYPE` | `contentType` not an allowed image/video type |
| `FILE_TOO_LARGE` | Declared size exceeds the limit |

---

### BW. Mutation: `RefreshToken`

Exchange a valid refresh token for a new access token (with a rotated refresh token). Called by the client when the access token has expired.

**Auth:** Not required (the refresh token is supplied in the input)

**Operation:**
```graphql
mutation RefreshToken($input: RefreshTokenInput!) {
  refreshToken(input: $input) {
    accessToken
    refreshToken   # rotated — replace the stored one
    expiresIn
  }
}
```

**Variables:**
```json
{ "input": { "refreshToken": "eyJ..." } }
```

**Response `200 OK`:**
```json
{
  "data": {
    "refreshToken": {
      "accessToken": "eyJ...",
      "refreshToken": "eyJ...",
      "expiresIn": 3600
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `INVALID_REFRESH_TOKEN` | Token is malformed or already used/revoked |
| `REFRESH_TOKEN_EXPIRED` | Refresh token has expired → client must force a fresh login |

---

## Edge Cases & Notes

| Case | Expected Behaviour |
|------|--------------------|
| User closes app mid-OTP | Phone + OTP screen state is lost; user must restart from phone entry |
| OTP expired before entry | Show "Code expired" with Resend button immediately active |
| Tag already set, user tries to change | `400 TAG_ALREADY_SET`; hide tag field on Edit Profile for existing users |
| OAuth account already registered | Treated as login (`isNewUser=false`); profile setup skipped only if `requiresProfileSetup=false` |
| User skips avatar on profile setup | Default avatar (initials or placeholder) used until manually updated |
| User abandons profile setup, logs in again | `isNewUser=false` but `requiresProfileSetup=true` → routed back into Profile Setup to finish name + tag before entering the app |
| Redirect to Login (from any auth-required action) | The originating action/screen is remembered as a **return target**; after successful login/signup (and Profile Setup if needed), navigate **back to that target**. No return target (cold open on Login) → land on Explore |
| Access token expired during a session | Client calls `RefreshToken (BW)` with the stored refresh token; on `REFRESH_TOKEN_EXPIRED` → force a fresh login |
