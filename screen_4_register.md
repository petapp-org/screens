# Screen 4: Register / Login

## Overview

Entry point for unauthenticated users. Supports two registration/login methods:
1. **Phone number → OTP**
2. **Google / Apple / Zalo ID (OAuth)**

After first-time registration via any method, user is prompted to complete their profile (name, username, avatar) before proceeding to the app.

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
     └─> SendOtp mutation (AA) { phone: "+84901234567" }
           ├─ INVALID_PHONE → show inline error
           └─ 200 → navigate to OTP screen

2. OTP screen: 6-digit input + countdown timer (60s resend)
     └─> VerifyOtp mutation (AB) { phone, code, deviceId }
           ├─ INVALID_OTP → show error, allow retry
           ├─ TOO_MANY_ATTEMPTS → show "Too many attempts, try again later"
           └─ 200 → { accessToken, refreshToken, isNewUser, sessionId }
                 ├─ isNewUser=true (or user.username==null) → navigate to Profile Setup screen
                 └─ else → navigate to post-login target
                                (the return target, else Explore)

3. Resend OTP: available after 60s countdown
     └─> SendOtp mutation (AA) (same as step 1)
```

### Flow B — Google / Apple / Zalo OAuth

```
1. User taps "Sign in with Google" / "Sign in with Apple"
     └─> Native OAuth flow (handled by SDK)
           └─> SocialLogin mutation (AC) { provider, token, deviceId }
                 └─ 200 → { accessToken, refreshToken, isNewUser, sessionId }
                       ├─ isNewUser=true (or user.username==null) → navigate to Profile Setup screen
                       └─ else → navigate to post-login target
                                      (the return target, else Explore)
```

---

> **Routing on `isNewUser` and `username`:** After authentication, routing uses `isNewUser` (and `user.username == null` for users who abandoned setup). `isNewUser` is just analytics (was the account created on this call). A user who registered but abandoned setup still has `username=null` → routed back into Profile Setup (even when `isNewUser=false`).

> **Return target (post-login redirect):** When the user reaches this screen because an auth-required action redirected them (e.g. tapped Follow / Message / My Pets), that origin is remembered as a **return target**. After successful auth — and after Profile Setup if `isNewUser=true` or `username==null` — the app navigates **back to the return target**. If there is no return target (app opened cold on Login), it lands on **Explore**.

> Backend does not return `requiresProfileSetup`. The client derives "needs Profile Setup" from `isNewUser` (or `user.username == null`).

---

## Profile Setup Screen *(first-time only, after registration)*

Shown once after a new account is created via any method.

```
[Avatar picker — optional, can skip]
[Name input — required]
[Username input — required, unique, format: letters/numbers/underscore only, no spaces]
[Continue button]
```

| Field | Rules |
|-------|-------|
| `avatar` | Optional; user can upload or skip (default avatar used) |
| `name` | Required; max 50 chars |
| `username` | Required; unique across all users; only set once at creation — **cannot be changed after this step** |

**Username validation:**
- Real-time uniqueness check as user types (debounced 500ms)
- Uses `CheckUsername` query (see endpoint AF below)
- Show green checkmark if available, red X if taken
- Show a persistent helper text below the field: *"⚠️ Your username cannot be changed after you create your account."*

**Submit:**
- If an avatar was picked: upload it first via `SignUploadBatch (BV)` `{ items: [{ purpose: "USER_AVATAR", ... }] }` → use `list[0].publicUrl` (no AI scan — avatars are never scanned)
- `UpdateMyProfile mutation (AE)` `{ displayName, username, avatarUrl }`
- On success → navigate to the **post-login target** (return target if redirected here, else Explore)

---

## API Endpoints Required

All operations use `POST /graphql`. Auth where required via `Authorization: Bearer <token>` header.

**Shared types:**
```graphql
type AuthTokens {
  accessToken: String!
  refreshToken: String!
  accessTokenExpiresIn: Int!
  sessionId: ID!
  user: User!
  isNewUser: Boolean!
}

type User {
  id: ID!
  displayName: String!
  username: String          # null until the user sets it during Profile Setup
  avatarUrl: String!
}
```

> Backend does not return `requiresProfileSetup`. The client derives "needs Profile Setup" from `isNewUser` (or `user.username == null`).

---

### AA. Mutation: `SendOtp`

Request an OTP be sent to the given phone number or email.
**Auth:** Not required

**Operation:**
```graphql
mutation SendOtp($phone: String) {
  sendOtp(phone: $phone) {
    otpRequestId
    expiresInSeconds
  }
}
```

**Variables:**
```json
{ "phone": "+84901234567" }
```

Note: pass either `phone` OR `email` — exactly one.

**Response `200 OK`:**
```json
{
  "data": {
    "sendOtp": {
      "otpRequestId": "otpreq_abc123",
      "expiresInSeconds": 60
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
mutation VerifyOtp($code: String!, $deviceId: String!, $phone: String) {
  verifyOtp(code: $code, deviceId: $deviceId, phone: $phone) {
    accessToken
    refreshToken
    accessTokenExpiresIn
    sessionId
    isNewUser
    user { id displayName username avatarUrl }
  }
}
```

**Variables:**
```json
{ "code": "123456", "deviceId": "dev_ios_abc", "phone": "+84901234567" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "verifyOtp": {
      "accessToken": "eyJ...",
      "refreshToken": "eyJ...",
      "accessTokenExpiresIn": 900,
      "sessionId": "sess_abc",
      "isNewUser": true,
      "user": {
        "id": "u_abc123",
        "displayName": "",
        "username": null,
        "avatarUrl": ""
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

### AC. Mutation: `SocialLogin`

Authenticate via Google, Apple, or Zalo OAuth. Returns `AuthTokens`.
_Thay cho AuthWithGoogle (AC) + AuthWithApple (AD)._
**Auth:** Not required

**Operation:**
```graphql
mutation SocialLogin($provider: AuthProvider!, $token: String!, $deviceId: String!) {
  socialLogin(provider: $provider, token: $token, deviceId: $deviceId) {
    accessToken
    refreshToken
    accessTokenExpiresIn
    sessionId
    isNewUser
    user { id displayName username avatarUrl }
  }
}
```

`provider`: `GOOGLE` | `APPLE` | `ZALO`. `token` = OAuth idToken.

**Variables (Google):**
```json
{ "provider": "GOOGLE", "token": "eyJ...", "deviceId": "dev_ios_abc" }
```

**Variables (Apple):**
```json
{ "provider": "APPLE", "token": "eyJ...", "deviceId": "dev_ios_abc" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "socialLogin": {
      "accessToken": "eyJ...",
      "refreshToken": "eyJ...",
      "accessTokenExpiresIn": 900,
      "sessionId": "sess_abc",
      "isNewUser": false,
      "user": {
        "id": "u_abc123",
        "displayName": "Minh Dang",
        "username": "minhdang",
        "avatarUrl": "https://..."
      }
    }
  }
}
```

---

### AE. Mutation: `UpdateMyProfile`

Update user profile fields (used for Profile Setup after first registration and for later edits).
**Auth:** Required

Same backend op as UpdateMe (screen_5) — both first-time Profile Setup and later edits use `updateMyProfile`.

**Operation:**
```graphql
mutation UpdateMyProfile($displayName: String, $username: String, $avatarUrl: String) {
  updateMyProfile(displayName: $displayName, username: $username, avatarUrl: $avatarUrl) {
    id
    displayName
    username
    avatarUrl
  }
}
```

**Variables:**
```json
{ "displayName": "Minh Dang", "username": "minhdang", "avatarUrl": "https://..." }
```

**Response `200 OK`:**
```json
{
  "data": {
    "updateMyProfile": {
      "id": "u_abc123",
      "displayName": "Minh Dang",
      "username": "minhdang",
      "avatarUrl": "https://..."
    }
  }
}
```

**Errors:**

| Code | Scenario |
|------|----------|
| `USERNAME_TAKEN` | Username already in use by another user |
| `USERNAME_ALREADY_SET` | User attempts to change username after it was already set |
| `INVALID_USERNAME_FORMAT` | Username contains invalid characters |

---

### AF. Query: `CheckUsername`

Check username availability during Profile Setup.
**Auth:** Not required

**Operation:**
```graphql
query CheckUsername($username: String!) {
  checkUsername(username: $username)
}
```

**Variables:**
```json
{
  "username": "minhdang"
}
```

**Response `200 OK`:**
```json
{
  "data": {
    "checkUsername": true
  }
}
```

Note: returns a Boolean scalar directly (no `{ available }` wrapper).

**Errors:**

| Status | Code | Scenario |
|--------|------|----------|
| `400` | `INVALID_USERNAME_FORMAT` | Username contains invalid characters |

---

### BV. Mutation: `SignUploadBatch`

Get pre-signed upload URLs for one or more media items. **Shared by all uploads** — user avatar, family avatar, pet avatar, and post media. **No AI scan** is performed here; scanning post media for pet detection is a separate, post-media-only step (`IdentifyPetFromMedia (AT)`, screen_7). Avatars are never scanned.

The mutation is a batch: wrap a single upload as a one-element `items` list and read the result from `list[0]`.

**Auth:** Required

**Operation:**
```graphql
mutation SignUploadBatch($items: [SignUploadBatchItemInput!]!) {
  signUploadBatch(items: $items) {
    uploadUrl      # pre-signed PUT URL — client uploads the raw bytes here
    publicUrl      # final hosted URL to store on the profile/family/pet/post
    expiresIn
  }
}
```

**`SignUploadBatchItemInput`:**
```graphql
input SignUploadBatchItemInput {
  ownerId: String!
  mediaType: MediaType!
  visibility: MediaVisibility!
  contentType: String!
  fileSizeBytes: Int!
  purpose: MediaPurposeGQL!
}
```

**Variables (single upload — wrap as one-element list):**
```json
{ "items": [{ "ownerId": "user_001", "mediaType": "IMAGE", "visibility": "PUBLIC", "contentType": "image/jpeg", "fileSizeBytes": 204800, "purpose": "USER_AVATAR" }] }
```

`purpose` enum: `USER_AVATAR` | `FAMILY_AVATAR` | `PET_AVATAR` | `POST_MEDIA`

**Client flow:** call this → `PUT` the file bytes to `uploadUrl` (from `list[0]`) → use `publicUrl` (from `list[0]`) in the next mutation (`UpdateMyProfile`, family/pet update, or `CreatePost`). For **post media only**, pass the media id to `IdentifyPetFromMedia (AT)` to detect/match a pet.

**Response `200 OK`:**
```json
{
  "data": {
    "signUploadBatch": [
      {
        "uploadUrl": "https://storage.petapp.com/upload/abc123?sig=...",
        "publicUrl": "https://cdn.petapp.com/media/abc123.jpg",
        "expiresIn": 300
      }
    ]
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
mutation RefreshToken($sessionId: String!, $refreshToken: String!, $deviceId: String!) {
  refreshToken(sessionId: $sessionId, refreshToken: $refreshToken, deviceId: $deviceId) {
    accessToken
    refreshToken   # rotated — replace the stored one
    accessTokenExpiresIn
    sessionId
  }
}
```

**Variables:**
```json
{ "sessionId": "sess_abc", "refreshToken": "eyJ...", "deviceId": "dev_ios_abc" }
```

**Response `200 OK`:**
```json
{
  "data": {
    "refreshToken": {
      "accessToken": "eyJ...",
      "refreshToken": "eyJ...",
      "accessTokenExpiresIn": 900,
      "sessionId": "sess_abc"
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
| Username already set, user tries to change | `400 USERNAME_ALREADY_SET`; hide username field on Edit Profile for existing users |
| OAuth account already registered | Treated as login (`isNewUser=false`); profile setup skipped only if `user.username != null` |
| User skips avatar on profile setup | Default avatar (initials or placeholder) used until manually updated |
| User abandons profile setup, logs in again | `isNewUser=false` but `user.username==null` → routed back into Profile Setup to finish name + username before entering the app |
| Redirect to Login (from any auth-required action) | The originating action/screen is remembered as a **return target**; after successful login/signup (and Profile Setup if needed), navigate **back to that target**. No return target (cold open on Login) → land on Explore |
| Access token expired during a session | Client calls `RefreshToken (BW)` with the stored refresh token; on `REFRESH_TOKEN_EXPIRED` → force a fresh login |
