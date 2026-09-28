---
title: "rollup"
id: "rollup"
version: "1.0.0"
status: "committed"
type: "artifact"
agent: "Business Analyst"
---

# Business Analysis — D-COPSCAR

## Discovery

**Brief Quality:** thin (Problem: strong, Outcomes: weak, Stakeholders: weak)

**Declaration:** Brief quality is thin on outcome measurability and stakeholder depth: the document specifies functional and API-level acceptance criteria (v1-clean and testable) but contains no explicit business KPI, target user count, success percentage, or timeframe, and it names delivery surfaces/roles rather than named stakeholders. Objectives and KPIs below are therefore derived from the acceptance criteria and in-scope behaviors as the closest measurable proxies for business outcomes; no external analytics, SLA targets, or user volumes were stated in the source and none have been invented.

## Business Objectives

### OBJ-001: Deliver a single shared backend credential-verification and JWT-issuance contract that both web and Android clients can authenticate against without divergence.
- **Leading KPI:** Percentage of `/auth/login` requests with correct credentials that return a 200 with valid `token` + `user` payload on first attempt
- **Lagging KPI:** Number of client-side auth integration defects/rework tickets attributable to backend contract mismatch, post-launch
- **Target:** 100% success rate for correct-credential logins across both clients within 60 days of shared backend go-live; zero contract-mismatch rework tickets in the same period
- **Impact:** Both client teams can build and ship protected features on top of a single, stable identity contract instead of each negotiating or duplicating auth logic.

### OBJ-002: Ensure protected endpoints uniformly reject unauthenticated or invalid-token requests, so gated functionality cannot be accessed without a valid session.
- **Leading KPI:** Percentage of protected-endpoint requests with missing/expired/invalid tokens that return 401
- **Lagging KPI:** Count of confirmed unauthorized-access incidents on protected endpoints post-launch
- **Target:** 100% of missing/expired/invalid-token requests to protected endpoints return 401, verified via test suite, at launch and sustained through 60 days with zero unauthorized-access incidents
- **Impact:** Protected functionality is reliably gated behind identity, establishing the security baseline all future protected features depend on.

### OBJ-003: Prevent credential-enumeration leakage by returning an identical error for unknown-email and wrong-password cases.
- **Leading KPI:** Percentage of failed-login responses (unknown email vs wrong password) returning an identical generic 401 message
- **Lagging KPI:** Number of confirmed instances (via code review or security testing) where the error response differs between the two failure cases, discovered post-launch
- **Target:** 100% of failed-login responses across both failure cases return the same generic message at launch; zero discrepancies found in security review within 60 days
- **Impact:** Removes a basic account-enumeration attack vector, reducing risk exposure for all existing account holders.

### OBJ-004: Enable existing users to authenticate consistently from either the web app or the Android app using the same credentials against the same backend, achieving cross-client parity.
- **Leading KPI:** Percentage of test accounts able to log in successfully with identical email/password on both web and Android
- **Lagging KPI:** Rate of cross-client login-parity support/bug reports (e.g., 'works on web, fails on Android' or vice versa) in the first 60 days post-launch
- **Target:** 100% cross-client login parity confirmed via acceptance testing at launch; zero cross-client parity defects reported in 60 days
- **Impact:** Existing users get one consistent identity across platforms, removing platform-specific friction and enabling future features to be built once against a shared identity model.

### OBJ-005: Ensure logout on either client fully revokes local access, preventing further authenticated calls until the user logs in again.
- **Leading KPI:** Percentage of post-logout protected API calls (using the discarded token) that fail with 401 in test scenarios
- **Lagging KPI:** Number of support/security reports of continued authenticated access after logout, within 60 days
- **Target:** 100% of post-logout protected calls return 401 in acceptance testing at launch; zero post-logout access reports within 60 days
- **Impact:** Users and stakeholders can trust that logout is a real security boundary on both clients, even without a server-side session store in v1.

## EP-001: Shared Backend Credential Verification & JWT Issuance (→ OBJ-001)
Build the shared backend's POST /auth/login endpoint: accept email+password, verify against existing accounts, and on success return a JWT plus minimal user identity (id, email). This is the single identity contract both web and Android clients depend on, so it must be built once and be authoritative for both surfaces - eliminating any risk of the two client teams negotiating or diverging on auth logic.

### US-001-01: As a Existing account holder logging in via web or Android app, I want to submit valid email and password to the shared backend and receive a JWT plus my user identity, so that so that I can authenticate successfully on the first attempt, directly driving the KPI of correct-credential requests returning 200 with valid token+user payload
**Priority:** Must Have

**Acceptance Criteria:**
- POST /auth/login with correct email+password returns HTTP 200
- Response body contains a valid JWT string in `token`
- Response body contains `user.id` and `user.email` matching the account
- No password or sensitive data is echoed in the response

**Test Cases:**
- **TC-001-01**: Valid credentials return 200 with token and user identity
- **TC-001-02**: Issued JWT is well-formed and usable

### US-001-02: As a Client developer (web or Android) integrating the login form, I want to receive a 400 response when submitting a missing or malformed request body, so that so that client-side validation errors are clearly distinguishable from auth failures, supporting a reliable first-attempt success contract measured by the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- POST /auth/login with missing email or password field returns HTTP 400
- POST /auth/login with malformed JSON body returns HTTP 400
- No JWT is issued on a 400 response

**Test Cases:**
- **TC-001-03**: Missing password field returns 400
- **TC-001-04**: Missing email field returns 400
- **TC-001-05**: Malformed JSON body returns 400
- **TC-001-06**: Empty request body returns 400

### US-001-03: As a Existing account holder who mistypes their email address, I want to receive a generic authentication error when the email does not match any account, so that so that failed login attempts are handled consistently and securely, supporting the KPI's integrity for first-attempt correctness reporting
**Priority:** Must Have

**Acceptance Criteria:**
- POST /auth/login with an unknown email returns HTTP 401
- No JWT is issued
- Error message does not reveal that the email is unrecognized

**Test Cases:**
- **TC-001-07**: Unknown email returns 401 with no token

### US-001-04: As a Existing account holder who enters the wrong password, I want to receive a generic authentication error when my password does not match my account, so that so that credential validation failures are handled securely without leaking account existence, protecting the KPI's measurement of legitimate first-attempt logins
**Priority:** Must Have

**Acceptance Criteria:**
- POST /auth/login with a valid email but wrong password returns HTTP 401
- No JWT is issued
- Error message does not reveal that the password was wrong specifically

**Test Cases:**
- **TC-001-08**: Wrong password returns 401 with no token

### US-001-05: As a Security-focused engineer validating the auth contract, I want to confirm that the error response for unknown email and wrong password is identical, so that so that the backend never leaks which credential failed, protecting account holders and preserving trust in the shared auth contract underpinning the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- The HTTP status code for unknown email and wrong password is identical (401)
- The error message/body content is identical for both failure cases
- No distinguishing field (timing, message, error code) reveals which check failed

**Test Cases:**
- **TC-001-09**: Unknown email and wrong password produce identical error bodies

### US-001-06: As a Web or Android client calling a protected API on behalf of an authenticated user, I want to successfully access a protected endpoint by presenting a valid Bearer JWT, so that so that authenticated sessions function correctly across both client surfaces, fulfilling the shared backend's contract measured by the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- A request to a protected endpoint with a valid, unexpired JWT in the Authorization header succeeds
- The backend correctly identifies the user from the token
- Access is granted regardless of which client (web or Android) issued the request

**Test Cases:**
- **TC-001-10**: Valid token grants access to protected endpoint

### US-001-07: As a Unauthenticated caller attempting to reach a protected endpoint, I want to receive a 401 error when no Authorization header is present, so that so that protected resources remain inaccessible without a valid session, reinforcing backend authority over auth enforcement per the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- A request to a protected endpoint without an Authorization header returns HTTP 401
- No resource data is returned in the response

**Test Cases:**
- **TC-001-11**: Missing Authorization header returns 401

### US-001-08: As a Client presenting an expired session token, I want to receive a 401 error when my JWT has expired, so that so that stale sessions cannot be used to access protected resources, upholding the integrity of the shared auth contract measured by the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- A request to a protected endpoint with an expired JWT returns HTTP 401
- No resource data is returned

**Test Cases:**
- **TC-001-12**: Expired token returns 401

### US-001-09: As a Client presenting a malformed or tampered token, I want to receive a 401 error when my JWT is invalid or corrupted, so that so that only cryptographically valid tokens issued by the shared backend can access protected resources, protecting the auth contract measured by the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- A request to a protected endpoint with a malformed or tampered JWT returns HTTP 401
- No resource data is returned
- Backend does not throw an unhandled/5xx error on malformed token input

**Test Cases:**
- **TC-001-13**: Malformed token string returns 401
- **TC-001-14**: Tampered signature returns 401

### US-001-10: As a Existing account holder using both the web app and the Android app, I want to log in with the same email and password on either client and get consistent results, so that so that the single shared backend behaves identically for both surfaces, eliminating divergence risk and satisfying the epic's objective of one authoritative auth contract
**Priority:** Must Have

**Acceptance Criteria:**
- Identical email/password payload sent from a simulated web client and a simulated Android client both return 200 with equivalent token and user payload structure
- No client-specific behavior differences exist in the backend's login response

**Test Cases:**
- **TC-001-15**: Same credentials succeed identically regardless of client origin

## EP-002: Protected Route Token Validation Middleware (→ OBJ-002)
Implement backend middleware/logic that inspects the Authorization: Bearer <jwt> header on every protected endpoint and rejects requests with a missing, expired, or invalid token with a 401. This establishes the uniform security gate that all current and future protected functionality relies on, independent of which client made the call.

### US-002-01: As a Backend platform engineer, I want to have the middleware reject any request to a protected endpoint that has no Authorization header at all, so that ensures unauthenticated requests are uniformly blocked with a 401, directly increasing the percentage of missing-token requests correctly rejected
**Priority:** Must Have

**Acceptance Criteria:**
- Any request to a protected endpoint without an Authorization header returns 401
- Response body/message does not leak internal details about why the request failed
- Behavior is identical regardless of which client (web or Android) made the call

**Test Cases:**
- **TC-002-01**: Missing Authorization header returns 401
- **TC-002-02**: Empty Authorization header value returns 401

### US-002-02: As a Backend platform engineer, I want to have the middleware reject requests whose Bearer token has expired, so that prevents access via stale tokens, keeping the 401-on-expired-token rate at 100% per the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- A syntactically valid JWT whose exp claim is in the past returns 401 on any protected endpoint
- The check runs on every protected endpoint invocation, not just at login

**Test Cases:**
- **TC-002-03**: Expired JWT returns 401
- **TC-002-04**: Token expiring exactly at request boundary is rejected

### US-002-03: As a Backend platform engineer, I want to have the middleware reject requests with a malformed, tampered, or otherwise invalid signature JWT, so that blocks forged or corrupted tokens from reaching gated functionality, supporting the KPI of 401 responses for invalid tokens
**Priority:** Must Have

**Acceptance Criteria:**
- A token with an invalid signature is rejected with 401
- A token that is not well-formed JWT (garbage string) is rejected with 401
- A token signed with a different/unknown key is rejected with 401

**Test Cases:**
- **TC-002-05**: Tampered payload signature mismatch returns 401
- **TC-002-06**: Non-JWT garbage string returns 401
- **TC-002-07**: Token signed with wrong key returns 401
- **TC-002-08**: Missing 'Bearer' scheme prefix returns 401

### US-002-04: As a Authenticated end user (web or Android app), I want to successfully access a protected endpoint when presenting a valid, unexpired Bearer token, so that confirms the gate does not falsely block legitimate sessions, which is essential to the KPI being measured correctly (only bad tokens are rejected, not good ones)
**Priority:** Must Have

**Acceptance Criteria:**
- A request with a valid, unexpired, correctly-signed JWT reaches the protected endpoint's business logic
- Response status is not 401 and reflects the endpoint's normal success behavior

**Test Cases:**
- **TC-002-09**: Valid token grants access to protected endpoint
- **TC-002-10**: Valid token works identically from web and Android clients

### US-002-05: As a Product/security stakeholder, I want to verify that the token-validation gate is applied uniformly across every protected endpoint in the system, not just a subset, so that guarantees no gated functionality can be reached without a valid session, ensuring the KPI holds system-wide rather than for only some endpoints
**Priority:** Must Have

**Acceptance Criteria:**
- Every endpoint designated as protected invokes the same middleware/logic for token validation
- No protected endpoint can be called successfully by bypassing the Authorization header check

**Test Cases:**
- **TC-002-11**: All protected endpoints enforce 401 on missing token
- **TC-002-12**: Newly added protected endpoint inherits middleware automatically

### US-002-06: As a Backend platform engineer, I want to validate tokens statelessly (signature + expiry check only) without requiring a server-side session store, so that keeps the validation gate lightweight and consistent with the v1 constraint of no server session store, while still reliably returning 401 for bad tokens per the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- Token validation does not query a session/session-store database; it relies on JWT signature verification and expiry claim only
- A logged-out client's previously issued but still-unexpired JWT is not rejected by the middleware itself (since v1 has no server-side logout blacklist) - logout is enforced client-side only

**Test Cases:**
- **TC-002-13**: Middleware validates without server session lookup
- **TC-002-14**: Still-unexpired token issued before client-side logout remains structurally valid at middleware level

## EP-003: Anti-Enumeration Generic Error Response (→ OBJ-003)
Standardize the /auth/login failure path so that unknown-email and wrong-password cases return the identical generic 401 error (with malformed/missing request bodies returning 400 separately). This closes the account-enumeration vector at the API layer, applying equally regardless of which client submitted the failed attempt.

### US-003-01: As a Backend engineer implementing /auth/login, I want to return the identical generic 401 error when a login attempt uses an email address that does not exist in the system, so that prevents an attacker from distinguishing 'unknown email' from other failures, directly raising the percentage of failed-login responses that return the identical generic 401 message (the epic KPI)
**Priority:** Must Have

**Acceptance Criteria:**
- POST /auth/login with a non-existent email returns HTTP 401
- Response body/message is byte-for-byte identical to the wrong-password failure response
- No token or user identity is included in the response
- Response does not contain any field or hint indicating the email was not found

**Test Cases:**
- **TC-003-01**: Unknown email returns generic 401
- **TC-003-02**: Unknown email response matches wrong-password response exactly

### US-003-02: As a Backend engineer implementing /auth/login, I want to return the identical generic 401 error when a login attempt uses a valid email but an incorrect password, so that prevents an attacker from confirming an email exists via a differing wrong-password error, contributing to the KPI of identical generic 401 responses across both failure cases
**Priority:** Must Have

**Acceptance Criteria:**
- POST /auth/login with a valid email and incorrect password returns HTTP 401
- Response body/message is byte-for-byte identical to the unknown-email failure response
- No token or user identity is included in the response

**Test Cases:**
- **TC-003-03**: Wrong password returns generic 401
- **TC-003-04**: Multiple wrong-password attempts return consistent error

### US-003-03: As a Backend engineer implementing /auth/login, I want to return HTTP 400 for a missing or malformed request body, distinct from the 401 anti-enumeration error, so that keeps validation errors separate from credential-failure errors so the 401 generic-error KPI is measured only against true unknown-email/wrong-password cases, not malformed requests
**Priority:** Must Have

**Acceptance Criteria:**
- Missing email field returns HTTP 400
- Missing password field returns HTTP 400
- Non-JSON or unparsable body returns HTTP 400
- 400 response is distinguishable from the 401 generic error and does not claim credentials were checked

**Test Cases:**
- **TC-003-05**: Missing password field returns 400
- **TC-003-06**: Malformed JSON body returns 400
- **TC-003-07**: Empty string email/password treated as validation failure, not enumeration case

### US-003-04: As a Security/QA tester auditing the login endpoint, I want to verify that failure responses for unknown-email and wrong-password cases are indistinguishable across headers, body, and status code, so that provides direct measurement of the KPI - percentage of failed-login responses returning an identical generic 401 message - by systematically comparing both failure paths
**Priority:** Must Have

**Acceptance Criteria:**
- Automated test suite compares status code, response body, and relevant headers between unknown-email and wrong-password responses
- No discrepancy in message content, error codes, or field-level hints is found
- Test is run against both web-originated and Android-originated requests to confirm client-agnostic behavior

**Test Cases:**
- **TC-003-08**: Cross-case response equality audit
- **TC-003-09**: Client-agnostic identical error verification

### US-003-05: As a Web app end user attempting to log in, I want to see a single generic error message on the login form regardless of whether I typed an unknown email or the wrong password, so that reinforces that the anti-enumeration protection is visible end-to-end on the web client, supporting the KPI at the surface where a real user would otherwise attempt to infer account existence
**Priority:** Must Have

**Acceptance Criteria:**
- Web login form displays the same generic error text for both unknown-email and wrong-password backend responses
- No client-side logic infers or displays which field was incorrect
- No token is stored or considered valid after a failed attempt

**Test Cases:**
- **TC-003-10**: Web UI shows identical error for unknown email
- **TC-003-11**: Web UI shows identical error for wrong password

### US-003-06: As a Android app end user attempting to log in, I want to see a single generic error message on the login screen regardless of whether I typed an unknown email or the wrong password, so that ensures the anti-enumeration protection is consistently surfaced on the Android client, supporting the KPI's applicability equally regardless of which client submitted the failed attempt
**Priority:** Must Have

**Acceptance Criteria:**
- Android login screen displays the same generic error text for both unknown-email and wrong-password backend responses
- No client-side logic infers or displays which field was incorrect
- No token is stored or considered valid after a failed attempt

**Test Cases:**
- **TC-003-12**: Android UI shows identical error for unknown email
- **TC-003-13**: Android UI shows identical error for wrong password

## EP-004: Web App Login Flow & Bearer Token Usage (→ OBJ-004)
Build the web login page (email, password, submit) that calls the shared backend, persists the returned JWT for the browser session, attaches it as a Bearer header on subsequent protected API calls, and redirects unauthenticated visits to protected routes back to login. This is the web user's primary workflow and must produce parity with Android against the same backend contract.

### US-004-01: As a Existing web app user with valid credentials, I want to submit my email and password on the web login page to sign in, so that I can authenticate successfully against the shared backend, contributing to the KPI of successful logins with identical credentials across clients
**Priority:** Must Have

**Acceptance Criteria:**
- Login page displays email, password fields, and a submit button
- On submit, the web app calls POST /auth/login with the entered email and password
- On 200 response, the JWT and user identity (id, email) are received and stored
- User is navigated to the authenticated area of the app after successful login

**Test Cases:**
- **TC-004-01**: Successful login with valid credentials
- **TC-004-02**: Login form fields are present and enabled

### US-004-02: As a Existing web app user who mistypes their password, I want to receive a clear but generic error when my password is wrong or my email is unknown, so that I understand my login failed without the system leaking which credential was incorrect, preserving parity with Android's error handling behavior tied to the KPI's cross-client consistency
**Priority:** Must Have

**Acceptance Criteria:**
- Wrong password for a valid email returns 401 with a generic error message
- Unknown email returns 401 with the same generic error message as wrong password
- No JWT is issued or stored on failure
- User remains on the login page after failure

**Test Cases:**
- **TC-004-03**: Wrong password shows generic error
- **TC-004-04**: Unknown email shows same generic error as wrong password

### US-004-03: As a Web app user who submits an incomplete or malformed login request, I want to receive a validation error when I submit missing or invalid email/password data, so that I get immediate, correct feedback distinct from authentication failure, supporting reliable login success measurement for the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- Submitting with missing email and/or password triggers a 400 response from the backend or client-side validation preventing submission
- Malformed request body results in 400, distinct from the 401 credential-failure case
- No JWT is issued on 400

**Test Cases:**
- **TC-004-05**: Empty email and password submission
- **TC-004-06**: Malformed email format submission

### US-004-04: As a Authenticated web app user browsing during a session, I want to have my JWT persisted for the duration of my browser session after logging in, so that I remain authenticated across page interactions in the session without re-entering credentials, matching the parity requirement with Android's secure storage behavior
**Priority:** Must Have

**Acceptance Criteria:**
- JWT is stored client-side using the product default (client-held Bearer JWT) immediately after successful login
- JWT remains available for reuse across subsequent requests within the same browser session
- JWT is not persisted beyond the browser session (e.g., cleared on browser/tab close if using session-scoped storage)

**Test Cases:**
- **TC-004-07**: JWT available for reuse after login within session
- **TC-004-08**: JWT does not persist beyond session scope

### US-004-05: As a Authenticated web app user calling a protected API, I want to have my stored JWT automatically attached as a Bearer token on subsequent protected API calls, so that I can access protected resources seamlessly after login, directly validating the KPI's requirement that login produces a usable token on protected endpoints
**Priority:** Must Have

**Acceptance Criteria:**
- All calls to protected endpoints include an Authorization header formatted as 'Bearer <jwt>'
- The correct, currently stored JWT is used for each request
- Requests succeed against the shared backend when the JWT is valid

**Test Cases:**
- **TC-004-09**: Protected API call includes Bearer header with valid JWT

### US-004-06: As a Unauthenticated visitor attempting to reach a protected route, I want to be redirected to the login page when I try to access a protected route without a valid session, so that Unauthorized access is prevented client-side, supporting consistent access control parity with the Android app per the epic objective
**Priority:** Must Have

**Acceptance Criteria:**
- Any direct navigation to a protected route without a valid JWT redirects to the login page
- No protected content or data is rendered prior to redirect
- After successful login, user can access the originally requested protected route (if applicable) or a default authenticated page

**Test Cases:**
- **TC-004-10**: Redirect to login on direct protected route access
- **TC-004-11**: No protected content flashes before redirect

### US-004-07: As a Authenticated web app user calling protected APIs with an expired or invalid token, I want to receive a 401 response and be handled appropriately when my stored token is expired or invalid, so that The system correctly enforces token validity, ensuring the login/session lifecycle behaves consistently as measured by the KPI's cross-client success criteria
**Priority:** Must Have

**Acceptance Criteria:**
- A protected API call with an expired token returns 401 from the shared backend
- A protected API call with a malformed/invalid token returns 401
- A protected API call with no token returns 401
- The web app responds to a 401 by treating the user as unauthenticated (e.g., redirecting to login)

**Test Cases:**
- **TC-004-12**: Expired token returns 401
- **TC-004-13**: Invalid/malformed token returns 401
- **TC-004-14**: Missing token returns 401

### US-004-08: As a Authenticated web app user who wants to end their session, I want to log out and have my locally stored JWT discarded, so that I can securely end my session on the web client, satisfying the acceptance criterion that logout prevents further authenticated calls until re-login
**Priority:** Must Have

**Acceptance Criteria:**
- Clicking logout clears the locally stored JWT immediately
- No server-side session invalidation call is required (client-side only for v1)
- After logout, the user is navigated away from authenticated views (e.g., to login or public page)

**Test Cases:**
- **TC-004-15**: Logout clears stored JWT

### US-004-09: As a Web app user who has just logged out, I want to be blocked from making further authenticated calls until I log in again, so that The session termination is enforced end-to-end, directly validating the acceptance criterion on logout behavior and supporting the KPI's login/logout success measurement
**Priority:** Must Have

**Acceptance Criteria:**
- After logout, any attempt to call a protected endpoint fails (no valid token available to attach)
- Any attempt to access a protected route after logout redirects to login
- Logging in again restores full authenticated access

**Test Cases:**
- **TC-004-16**: Protected API call blocked after logout
- **TC-004-17**: Re-login after logout restores access

### US-004-10: As a QA engineer validating cross-client credential parity, I want to verify that the same email/password combination succeeds on the web app exactly as it does on the Android app against the shared backend, so that This directly measures the epic's KPI: percentage of test accounts able to log in successfully with identical email/password on both web and Android
**Priority:** Must Have

**Acceptance Criteria:**
- A given test account's email/password combination results in a successful 200 login response with token on the web app
- The identical credentials, when used against the shared backend from the Android app, also succeed
- Both clients receive equivalent user identity data (id, email) in the login response

**Test Cases:**
- **TC-004-18**: Cross-client login parity with same credentials

## EP-005: Android App Login Flow & Bearer Token Usage (→ OBJ-004)
Build the Android login screen (email, password, submit) that calls the shared backend, stores the returned JWT in app-private secure storage (e.g. EncryptedSharedPreferences/Keystore), attaches it as a Bearer header on subsequent protected API calls, and navigates unauthenticated access attempts on protected screens back to login. This is the Android user's primary workflow and must produce parity with web against the same backend contract.

### US-005-01: As a Existing Android app user, I want to view and use a login screen with email and password fields and a submit button, so that so I can initiate authentication with the same credentials I use on web, supporting cross-client parity for the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- Login screen displays email input, password input (masked), and a submit control
- Submit is disabled or validated when fields are empty
- Submitting triggers a call to POST /auth/login on the shared backend

**Test Cases:**
- **TC-005-01**: Login screen renders required fields
- **TC-005-02**: Submit disabled/blocked with empty fields

### US-005-02: As a Existing Android app user, I want to submit valid email and password and have the app call the shared backend to authenticate, so that so I can log in successfully using the same credentials that work on web, directly supporting the cross-client login parity KPI
**Priority:** Must Have

**Acceptance Criteria:**
- Valid credentials submitted via POST /auth/login
- 200 response containing token and user (id, email) is handled by the client
- User is navigated to the authenticated area of the app on success

**Test Cases:**
- **TC-005-03**: Successful login with valid credentials
- **TC-005-04**: Cross-client credential parity

### US-005-03: As a Existing Android app user, I want to receive a generic error when submitting wrong password or unknown email, so that so I understand my login failed without the app leaking which credential was incorrect, per the shared backend contract
**Priority:** Must Have

**Acceptance Criteria:**
- 401 response from backend for wrong password or unknown email is displayed as a single generic error message
- No JWT is stored and user remains on login screen
- Error message text is identical regardless of which credential was wrong

**Test Cases:**
- **TC-005-05**: Wrong password shows generic error
- **TC-005-06**: Unknown email shows same generic error

### US-005-04: As a Existing Android app user, I want to receive a validation error when submitting a missing or malformed login request, so that so I get clear feedback to correct my input before authentication is attempted, ensuring reliable login flow parity
**Priority:** Must Have

**Acceptance Criteria:**
- 400 response from backend for missing/invalid body is surfaced to the user
- No JWT stored, user remains on login screen

**Test Cases:**
- **TC-005-07**: Missing password field submission
- **TC-005-08**: Malformed email format submission

### US-005-05: As a Existing Android app user, I want to have the JWT stored securely in app-private storage after successful login, so that so my credentials/session remain protected on-device, supporting secure cross-client parity with web's token handling
**Priority:** Must Have

**Acceptance Criteria:**
- JWT is persisted using EncryptedSharedPreferences or Keystore-backed secure storage
- Token is not stored in plaintext or accessible outside the app sandbox
- Token persists across app restarts within the session validity

**Test Cases:**
- **TC-005-09**: JWT stored in encrypted storage after login
- **TC-005-10**: Stored token survives app restart

### US-005-06: As a Existing Android app user, I want to have the app attach my stored JWT as a Bearer header on subsequent protected API calls, so that so I can access protected resources after login without re-authenticating each request, matching the web client's behavior
**Priority:** Must Have

**Acceptance Criteria:**
- Every authenticated API request includes 'Authorization: Bearer <jwt>' header
- Header uses the exact token returned at login without modification

**Test Cases:**
- **TC-005-11**: Bearer header attached on protected API call

### US-005-07: As a Existing Android app user, I want to be navigated to the login screen when I try to access a protected screen without a valid token, so that so unauthenticated access is prevented on Android, matching the web app's redirect-to-login behavior for parity
**Priority:** Must Have

**Acceptance Criteria:**
- Attempting to open a protected screen with no stored token navigates to login
- Attempting to open a protected screen with an expired/invalid token navigates to login after a 401 is received

**Test Cases:**
- **TC-005-12**: No token, protected screen access blocked
- **TC-005-13**: Expired/invalid token triggers redirect to login

### US-005-08: As a Existing Android app user, I want to log out and have my stored JWT discarded, so that so I can end my session and prevent further authenticated calls until I log in again, matching web logout parity
**Priority:** Must Have

**Acceptance Criteria:**
- Logout action clears JWT from secure storage
- After logout, protected screens are inaccessible without re-authenticating
- No server-side session invalidation call is required (client-only discard)

**Test Cases:**
- **TC-005-14**: Logout clears stored JWT
- **TC-005-15**: Post-logout protected access requires re-login

## EP-006: Web App Logout & Local Session Termination (→ OBJ-005)
Implement web logout that discards the locally stored JWT (no server-side session store required), so that any subsequent protected API call from that browser session fails until the user logs in again. This gives web users a trustworthy security boundary at logout.

### US-006-01: As a authenticated web app user, I want to click a logout control to discard my locally stored JWT, so that so that my session is fully terminated on this browser, directly supporting the KPI of post-logout protected calls failing with 401
**Priority:** Must Have

**Acceptance Criteria:**
- Logout control is visible/available only when a JWT is present in client storage
- Clicking logout removes the JWT from wherever it is held (memory, sessionStorage, or httpOnly cookie wrapper) with no server call required
- After logout, the UI reflects a logged-out state (e.g., redirected to login page)
- No server-side session store is created or relied upon for this action

**Test Cases:**
- **TC-006-01**: Logout clears stored JWT from client storage
- **TC-006-02**: Logout redirects user to login page

### US-006-02: As a authenticated web app user, I want to attempt a protected API call using the token that existed before I logged out, so that so I can be confident logout revokes access, satisfying the KPI that post-logout calls with the discarded token fail with 401
**Priority:** Must Have

**Acceptance Criteria:**
- After logout, any attempt to call a protected endpoint using the previously stored JWT results in a 401 response
- This holds true even though no server-side session store is used - the client no longer sends a valid Bearer token because it has discarded it
- If a copy of the old token is manually replayed against a protected endpoint after logout, backend token validation (expired/invalid) rules still apply as per shared backend contract

**Test Cases:**
- **TC-006-03**: Protected API call fails after logout using captured old token
- **TC-006-04**: UI-triggered protected API call after logout returns 401 due to missing token

### US-006-03: As a logged-out web app visitor, I want to navigate directly to a protected route after logging out, so that so that the app enforces the security boundary and does not expose protected content post-logout, supporting the objective of fully revoking local access
**Priority:** Must Have

**Acceptance Criteria:**
- After logout, navigating (via URL bar, browser back button, or bookmark) to any protected route redirects the user to the login page
- No protected page content or data is rendered before or during the redirect

**Test Cases:**
- **TC-006-05**: Direct URL navigation to protected route after logout
- **TC-006-06**: Browser back button to protected route after logout

### US-006-04: As a returning web app user, I want to log in again after having previously logged out, so that so that logout does not permanently lock me out and the login/logout cycle works repeatedly, ensuring the security boundary is a clean toggle rather than a broken state
**Priority:** Must Have

**Acceptance Criteria:**
- After logout, submitting valid email/password on the login form succeeds and issues a new JWT
- The newly issued JWT allows successful calls to protected endpoints
- The user is not blocked or rate-limited solely due to having logged out previously

**Test Cases:**
- **TC-006-07**: Re-login succeeds after prior logout
- **TC-006-08**: New JWT after re-login works on protected endpoint

### US-006-05: As a web app user, I want to click logout when no active session/token exists (e.g., double-click or stale UI state), so that so that the logout action is idempotent and does not cause errors, keeping the security boundary reliable regardless of edge-case timing
**Priority:** Should Have

**Acceptance Criteria:**
- If logout is triggered while no JWT is present in storage, the app handles this gracefully (no crash, no error thrown to the user)
- The user ends up on the login page regardless of whether a token existed
- No server error is required or expected since v1 has no server-side session store

**Test Cases:**
- **TC-006-09**: Logout action with no stored JWT present

### US-006-06: As a security-conscious web app user, I want to have my JWT discarded regardless of which storage mechanism the app uses (memory, sessionStorage, or httpOnly cookie wrapper), so that so that logout reliably terminates my session no matter the underlying implementation choice, ensuring consistent 401 outcomes as measured by the KPI
**Priority:** Must Have

**Acceptance Criteria:**
- Logout clears the JWT correctly whether it is held in memory, sessionStorage, or an httpOnly cookie wrapping the token
- If httpOnly cookie storage is used, logout triggers an appropriate mechanism (e.g., a call that expires/clears the cookie) since JavaScript cannot directly read/clear httpOnly cookies
- Post-logout, no valid Bearer token is available to the client for any storage mechanism chosen

**Test Cases:**
- **TC-006-10**: Logout clears JWT held in sessionStorage
- **TC-006-11**: Logout clears JWT held in memory
- **TC-006-12**: Logout clears JWT held in httpOnly cookie wrapper

## EP-007: Android App Logout & Local Session Termination (→ OBJ-005)
Implement Android logout that discards the locally stored JWT from secure storage, so that any subsequent protected API call from that device fails until the user logs in again. This gives Android users a trustworthy security boundary at logout, matching the web behavior.

### US-007-01: As a Android app user with an active session, I want to tap the logout control to discard my locally stored JWT, so that so my device no longer holds a valid credential, directly reducing the rate of post-logout protected calls that succeed (supporting the KPI of 401s after logout)
**Priority:** Must Have

**Acceptance Criteria:**
- Tapping logout removes the JWT from the app's secure storage (EncryptedSharedPreferences/Keystore-backed store)
- No server session store is contacted or required to complete logout
- The logout action completes without error even if the network is unavailable

**Test Cases:**
- **TC-007-01**: Logout clears JWT from secure storage
- **TC-007-02**: Logout succeeds while device is offline

### US-007-02: As a Android app user who just logged out, I want to have any subsequent protected API call from my device rejected, so that so logout provides a trustworthy security boundary, measured directly by the KPI of post-logout protected calls failing with 401
**Priority:** Must Have

**Acceptance Criteria:**
- After logout, the app makes no further protected API calls using the discarded token
- If a protected call is attempted (e.g., via a stale in-flight request or manual replay) with the discarded token, the shared backend returns 401
- The app does not silently retry or refresh an invalid token after logout

**Test Cases:**
- **TC-007-03**: Replaying discarded token against protected endpoint returns 401
- **TC-007-04**: App does not send stale token on background refresh after logout

### US-007-03: As a Android app user who has logged out, I want to be navigated to the login screen when I try to access a protected screen, so that so I cannot view authenticated content after logout, reinforcing the security boundary the KPI measures
**Priority:** Must Have

**Acceptance Criteria:**
- Attempting to open any protected screen after logout redirects the user to the login screen
- No cached authenticated content is rendered before the redirect completes
- Login screen is presented with empty email/password fields

**Test Cases:**
- **TC-007-05**: Protected screen navigation redirects to login after logout
- **TC-007-06**: No cached authenticated data flashes before redirect

### US-007-04: As a Android app user who has logged out, I want to log back in with my email and password to regain authenticated access, so that so logout only blocks access until I re-authenticate, consistent with the objective that logout prevents further authenticated calls until login again
**Priority:** Must Have

**Acceptance Criteria:**
- After logout, submitting valid credentials on the login screen succeeds and issues a new JWT
- The new JWT is stored in secure storage and used for subsequent protected calls
- Protected API calls made with the new token succeed (do not return 401)

**Test Cases:**
- **TC-007-07**: Re-login after logout restores protected access
- **TC-007-08**: New token after re-login differs from previously discarded token

### US-007-05: As a Android app user who is already logged out, I want to tap logout again without causing an error, so that so the logout control remains a reliable, trustworthy action in all states, supporting consistent 401 enforcement measured by the KPI
**Priority:** Should Have

**Acceptance Criteria:**
- Tapping logout when no JWT is stored does not crash or throw a visible error
- The app remains on or returns to the login screen
- Secure storage remains empty of any JWT after the action

**Test Cases:**
- **TC-007-09**: Logout is a safe no-op with no active session

### US-007-06: As a Android app user who logs out and force-closes the app, I want to have my session remain terminated after the app is killed and reopened, so that so logout is a durable security boundary and not just an in-memory state, ensuring the KPI holds even across app restarts
**Priority:** Must Have

**Acceptance Criteria:**
- After logout, force-closing and reopening the app does not restore the discarded JWT
- Reopening the app after logout presents the login screen, not a protected screen
- No protected API call succeeds using any token cached in memory prior to the force-close

**Test Cases:**
- **TC-007-10**: Session stays terminated after app restart post-logout
- **TC-007-11**: Protected call fails after restart with no re-login

## Business Rules

### BR-001: The shared backend is the single authoritative source for credential verification and JWT issuance; no client (web or Android) may implement its own credential-checking or token-issuing logic.
**Rationale:** Prevents divergence between clients and guarantees cross-client login parity, which is the measured KPI for OBJ-001 and OBJ-004. | **Enforcement:** POST /auth/login is the only credential-verification endpoint; clients must call it rather than validating locally.

### BR-002: A successful login (correct email + password) must return HTTP 200 with a JWT (`token`) and minimal user identity (`user.id`, `user.email`) only - no password or other sensitive fields may be echoed.
**Rationale:** Defines the exact success contract both clients depend on and prevents accidental sensitive-data leakage. | **Enforcement:** Response schema validation on /auth/login; automated tests assert absence of password fields.

### BR-003: A login attempt with an unknown email and a login attempt with a valid email but wrong password must produce byte-for-byte identical HTTP 401 responses (status, body, and relevant headers).
**Rationale:** Eliminates account-enumeration attack vector; core anti-enumeration control for OBJ-003. | **Enforcement:** Shared error-handling code path in /auth/login for both failure branches; automated diff tests (TC-001-09, TC-003-02, TC-003-08).

### BR-004: A missing or malformed request body (missing email/password field, unparsable JSON, empty strings) must return HTTP 400, and must never be treated as, or produce the same response as, a 401 credential failure.
**Rationale:** Keeps validation errors distinct from authentication failures so the 401 anti-enumeration KPI is measured only against true credential-failure cases. | **Enforcement:** Input validation layer runs before credential lookup in /auth/login; 400 responses issued prior to any account lookup.

### BR-005: No JWT is issued under any failure condition (400 or 401) - a `token` field must never appear in a non-200 response body.
**Rationale:** Prevents partial-success ambiguity and guarantees only successful authentications produce usable credentials. | **Enforcement:** Response contract enforced at the API layer; tested across TC-001-03 to TC-001-09.

### BR-006: Every protected endpoint must require a valid, unexpired, correctly-signed JWT presented as `Authorization: Bearer <jwt>`; requests missing this header, with an expired token, or with a malformed/tampered/mis-signed token must return HTTP 401.
**Rationale:** Establishes the uniform security gate all current and future protected functionality depends on. | **Enforcement:** Centralized authentication middleware applied to all protected routes via standard route-registration pattern (verified in TC-002-11, TC-002-12).

### BR-007: Token validation must be stateless: performed via JWT signature verification and expiry (`exp`) claim check only, with no server-side session-store lookup.
**Rationale:** Matches the explicit v1 constraint of no server-side session store while still reliably returning 401 for bad tokens. | **Enforcement:** Middleware implementation restricted to cryptographic/claim checks; verified via absence of session-store queries in backend traces (TC-002-13).

### BR-008: A token whose `exp` claim is at or before the current server time must be treated as expired and rejected with 401, including exact-boundary cases.
**Rationale:** Removes ambiguity at the expiry boundary so stale sessions cannot be used. | **Enforcement:** Expiry comparison logic in token-validation middleware (TC-002-04).

### BR-009: Logout is a client-side-only operation: it discards the locally stored JWT and requires no server-side call, session invalidation, or blacklist entry in v1.
**Rationale:** Consistent with the explicit v1 scope decision of no server-side session store; keeps logout lightweight and available offline. | **Enforcement:** Web/Android logout handlers clear local storage (sessionStorage/memory/httpOnly cookie wrapper on web; EncryptedSharedPreferences/Keystore on Android) without a network call.

### BR-010: Logout must be idempotent: triggering logout when no JWT is present must not error, and must leave the user on/redirected to the login state.
**Rationale:** Guards against edge-case timing (double-click, stale UI) undermining trust in the security boundary. | **Enforcement:** Logout handler checks for existing token defensively before clearing; no exception thrown if absent (US-006-05, US-007-05).

### BR-011: After logout, the client must not attach any Authorization header (valid or otherwise) to subsequent requests, and any protected route/screen access attempt must redirect/navigate to login before rendering protected content.
**Rationale:** Enforces the end-to-end security boundary at the UI layer, not just the network layer. | **Enforcement:** Client-side route guards on web (redirect) and Android (navigation) triggered by absence of a stored token or a 401 response.

### BR-012: Re-login after logout must succeed with valid credentials and issue a fresh JWT; logout must never permanently or temporarily block future login attempts.
**Rationale:** Logout must be a reversible toggle, not a broken/locked state. | **Enforcement:** No account-level flag or rate limit is set by logout; /auth/login remains fully available immediately after logout (TC-006-07, TC-007-07).

### BR-013: The Android app must persist the JWT only in app-private secure storage (EncryptedSharedPreferences or Keystore-backed store) - never in plaintext files, logs, or unencrypted preferences.
**Rationale:** Protects the session credential from extraction outside the app sandbox, per explicit client requirement. | **Enforcement:** Android storage layer uses only encrypted/Keystore-backed APIs; verified via storage inspection in debug/test harness (TC-005-09).

### BR-014: The web app must persist the JWT using the product default of a client-held Bearer JWT (memory, sessionStorage, or an httpOnly-cookie wrapper), scoped to the browser session, and must not persist it beyond the session in session-scoped storage modes.
**Rationale:** Provides parity with Android's secure-storage intent while leaving the exact mechanism as an implementation choice, per source document. | **Enforcement:** Web token-storage module implements one of the three approved mechanisms; whichever is chosen must still be clearable in full on logout (BR-009).

### BR-015: Unauthenticated access to any protected route/screen (direct navigation, deep link, back button, or bookmark) must redirect/navigate to the login page/screen without rendering any protected content, even momentarily.
**Rationale:** Prevents any flash-of-protected-content security leak on both client surfaces. | **Enforcement:** Route guard executes before render/mount of protected views on both web (routing layer) and Android (navigation layer).

### BR-016: The feature scope is strictly limited to login and logout with email+password; account registration, password reset/change, OAuth/SSO/social login, iOS, refresh tokens/sliding sessions, roles, MFA, lockout policy, and audit UI are explicitly out of scope for v1 and must not be implemented as part of this feature.
**Rationale:** Keeps delivery bounded to the documented v1 scope and prevents scope creep into adjacent, undocumented auth capabilities. | **Enforcement:** Code review / acceptance criteria checklist against the source document's 'Out of scope' list before release sign-off.

### BR-017: The identical generic 401 error behavior for unknown-email and wrong-password cases must be verified and surfaced consistently regardless of which client (web or Android) or which User-Agent originates the request.
**Rationale:** The anti-enumeration guarantee is a backend contract property and must not vary based on caller identity or client metadata. | **Enforcement:** Backend error-handling logic contains no branching on client-identifying request metadata (User-Agent, client ID, etc.) for the login-failure path.

## Non-Functional Requirements

### NFR-001: POST /auth/login must respond within a bounded time for both success and failure paths under normal load.
**Category:** Performance | **Target:** p95 response time ≤ 500ms for /auth/login under expected concurrent load (baseline: 50 concurrent requests/sec) | **Priority:** Should Have

### NFR-002: Protected-endpoint token validation middleware must add minimal latency overhead since it runs on every gated request.
**Category:** Performance | **Target:** Token validation adds ≤ 10ms p95 overhead per request | **Priority:** Should Have

### NFR-003: All credential and token traffic (login requests, Bearer-authenticated calls) must be transmitted over TLS/HTTPS only; plaintext HTTP must be rejected or redirected.
**Category:** Security | **Target:** 100% of /auth/login and protected-endpoint traffic served over TLS 1.2+ in all environments | **Priority:** Must Have

### NFR-004: Stored passwords must be hashed with a strong, salted, adaptive hashing algorithm; plaintext passwords must never be stored or logged.
**Category:** Security | **Target:** 100% of stored credentials use bcrypt/argon2 (or equivalent) with per-record salt; zero plaintext password occurrences in logs or DB, verified via code/security review | **Priority:** Must Have

### NFR-005: JWTs must be signed with a strong algorithm and secret/key not exposed to clients; tokens signed with an unknown/wrong key must be rejected.
**Category:** Security | **Target:** 100% of tokens signed with HS256/RS256 (or equivalent) using a server-managed secret/key; 100% rejection rate for mis-signed tokens in test suite (TC-002-07) | **Priority:** Must Have

### NFR-006: The anti-enumeration guarantee (identical 401 for unknown email vs wrong password) must hold with zero discrepancies across status code, body, and headers.
**Category:** Security | **Target:** 0 discrepancies found across automated diff tests (TC-001-09, TC-003-02, TC-003-08) at launch and in every regression run | **Priority:** Must Have

### NFR-007: No malformed or tampered token input may cause an unhandled exception or 5xx server error; all invalid-token paths must resolve to a handled 401.
**Category:** Security | **Target:** 0 occurrences of 5xx responses across all malformed/tampered/garbage-token test cases (TC-001-13, TC-002-05, TC-002-06) | **Priority:** Must Have

### NFR-008: Android-stored JWTs must reside only in encrypted, app-private storage (EncryptedSharedPreferences/Keystore-backed), never in plaintext files, external storage, or logs.
**Category:** Security | **Target:** 100% of storage inspection test passes (TC-005-09) confirming no plaintext token artifacts | **Priority:** Must Have

### NFR-009: The shared backend's /auth/login and token-validation middleware must be highly available since both client apps are fully blocked from all protected functionality if it is down.
**Category:** Availability | **Target:** 99.9% uptime for the shared backend auth endpoints measured monthly | **Priority:** Should Have

### NFR-010: The token-validation middleware must scale horizontally without requiring shared session state, consistent with its stateless design.
**Category:** Scalability | **Target:** Middleware sustains correct 401/200 behavior at 10x baseline concurrent request volume with no added latency degradation beyond NFR-002 targets, with zero shared-state dependency (no session-store calls, per BR-007) | **Priority:** Should Have

### NFR-011: Client-side logout must complete successfully even when the device/browser has no network connectivity, since it is a purely local operation.
**Category:** Availability | **Target:** 100% pass rate on offline-logout test cases (TC-007-02) across both clients | **Priority:** Must Have

### NFR-012: The generic authentication error message shown on both web and Android login UIs must be identical in wording across both clients and both failure cases (unknown email, wrong password).
**Category:** Usability | **Target:** 100% string-match on displayed error text across TC-003-10/11/12/13 test pairs | **Priority:** Must Have

### NFR-013: Login/credential-failure handling must not log or expose which specific field (email vs password) failed validation, in any client or server log output, to avoid indirect information disclosure.
**Category:** Compliance | **Target:** 0 log entries or API responses across security review sample that distinguish 'unknown email' from 'wrong password' outcomes | **Priority:** Must Have

### NFR-014: Cross-client behavior (login, protected access, logout) must be functionally identical for web and Android against the shared backend, with no platform-specific divergence in outcomes for identical inputs.
**Category:** Availability | **Target:** 100% pass rate on cross-client parity test suite (TC-001-15, TC-004-18, TC-005-04) at launch and in every regression cycle within 60 days | **Priority:** Must Have

### NFR-015: Post-logout, no client-initiated request (UI-triggered) may present a previously-valid Bearer token to a protected endpoint; the client must have fully discarded it prior to any further protected call.
**Category:** Security | **Target:** 100% of UI-triggered protected calls post-logout observed with no Authorization header (or return 401 if attempted) across TC-006-04, TC-007-03/04, TC-007-11 | **Priority:** Must Have

## Critique Summary

**Verdict:** ship

**Notes:**