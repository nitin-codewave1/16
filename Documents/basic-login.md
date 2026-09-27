# Basic Login — Requirements (v1)

Standalone feature. Login only (no signup, reset, or SSO).

## Goal

Let an existing user sign in with email and password from a **web app** or an **Android app**, using one **shared backend** that issues a JWT.

## Surfaces

| Surface | Role |
|---------|------|
| Web app | Login form; store JWT; call protected APIs with `Authorization: Bearer <token>` |
| Android app | Login screen; store JWT securely; same Bearer header on API calls |
| Shared backend | Validate credentials; issue JWT; reject invalid/missing tokens on protected routes |

## In scope

1. User submits **email** + **password**.
2. Backend verifies credentials against existing accounts.
3. On success: return a JWT (and minimal user identity, e.g. `userId`, `email`).
4. On failure: return a clear error (wrong credentials / validation).
5. Clients attach the JWT to subsequent API requests.
6. Protected endpoints return **401** when the token is missing, expired, or invalid.
7. Logout on both clients: discard the local JWT (no server session store required for v1).

## Out of scope (v1)

- Account registration / invite
- Password reset / change
- OAuth / SSO / social login
- iOS
- Refresh tokens / sliding sessions (optional later)
- Roles, MFA, lockout policy, audit UI

## Shared API (minimal)

### `POST /auth/login`

**Request**

```json
{
  "email": "user@example.com",
  "password": "••••••••"
}
```

**Success `200`**

```json
{
  "token": "<jwt>",
  "user": {
    "id": "…",
    "email": "user@example.com"
  }
}
```

**Failure**

| Case | Status |
|------|--------|
| Missing/invalid body | `400` |
| Unknown email or wrong password | `401` (same message for both — do not leak which failed) |

### Authenticated calls

- Header: `Authorization: Bearer <jwt>`
- Invalid/expired/missing token → `401`

## Client requirements

### Web

- Login page: email, password, submit.
- Persist JWT for the browser session (implementation choice: memory, `sessionStorage`, or httpOnly cookie wrapping the same token — product default: client-held Bearer JWT for parity with Android).
- Unauthenticated visit to a protected route → redirect to login.
- Logout clears stored JWT.

### Android

- Login screen: email, password, submit.
- Store JWT in app-private secure storage (e.g. EncryptedSharedPreferences / Keystore-backed store).
- Unauthenticated access to protected screens → navigate to login.
- Logout clears stored JWT.

## Acceptance criteria

- [ ] Same email/password works on web and Android against the same backend.
- [ ] Successful login returns a JWT usable on a protected endpoint from both clients.
- [ ] Wrong password (or unknown email) returns `401` with a generic error; no token issued.
- [ ] Request without a valid JWT to a protected endpoint returns `401`.
- [ ] Logout on either client prevents further authenticated calls until login again.

## Non-goals for this doc

UI polish, branding, analytics, and account-management flows.
