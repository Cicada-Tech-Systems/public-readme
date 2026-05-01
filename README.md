# HomeDots API Backend

Healthcare IoT backend for BedDot sensor devices. Provides device management, vital signs monitoring, user authentication, alert system, and mobile/web APIs.

---

## Migration Guide — What Changed

If your app was built against the previous API, review the items below. Items marked **BREAKING** require code changes.

### Breaking Changes

| # | Change | Details | Action Required |
|---|--------|---------|-----------------|
| 1 | **Role renamed** | `co-owner` → `manager` | Replace all `"co-owner"` strings with `"manager"` in invite/role API calls |
| 2 | **`/mobile/auth/verify-firebase-token` removed** | Firebase phone auth no longer supported | Use `/mobile/auth/google` or `/mobile/auth/apple` instead |
| 3 | **`/mobile/invite-user` body changed** | `invitee_email` was required, now optional | No action if you always send it. New: response includes `invite_token` field |
| 4 | **Error status codes reclassified** (2026-04-24) | Many client errors were returning `500` (the raw `AppError` default). They now return the correct 4xx class: `400` (validation), `401` (auth failure), `403` (permission), `404` (not found), `409` (conflict / duplicate). | Clients that branched on `500` as a catch-all for all failures must now handle specific 4xx codes. Messages are unchanged — the change is HTTP status only. |
| 5 | **Refresh-token rotation revokes the old access token** (2026-04-24) | Calling `/mobile/auth/refresh-token` used to return a new pair but the old access token stayed valid until its `exp`. It is now revoked server-side at rotation time. | After calling `/refresh-token`, replace BOTH stored tokens; requests with the previous access token return `401` even before `exp`. |
| 6 | **Auth rate limits raised** (2026-04-24) | All auth endpoints were uniformly `5/min`. Login bumped to `30/min`, verify-otp `15/min`, signup `10/min`. Email-sending endpoints (resend-otp, reset-password) stay at `5/min`. | None — clients only see fewer 429s. |
| 7 | **Email + MAC normalisation** (2026-04-25) | `email` / `invitee_email` / `device_mac` were stored verbatim. Server now lowercases + strips before any lookup or write; existing rows backfilled. | None — already-normalised input is a no-op. Mobile clients should still send the original case for display, but stop hand-rolling case-folding for the wire. |
| 8 | **Invite remove/role-change require ONE of target_user_id or target_email** (2026-04-25) | `RemoveDeviceUserBody` / `UpdateDeviceUserRoleBody` previously required `target_user_id: int`. Sending `target_user_id: null` for a pending-invite-to-unregistered-email failed with a generic 422 / 404. They now accept either field; clients must send exactly one. | **Yes** — clients sending `target_user_id: null` for unregistered invitees must switch to `target_email: <email>` (the email is already in the `get-device-users` response). |
| 9 | **Master/slave mode removed** (2026-04-29) | The `IS_MASTER_SERVER` dual-mode pattern, the `_post_to_master` slave-forwarding path, and the legacy `auth_user(user_name, user_token)` body-based auth path were all deleted. The backend is single-master now. JWT is the only mobile-auth mechanism. | Drop `IS_MASTER_SERVER`, `MASTER_SERVER_URL`, and the legacy backend `user_token` from your env / config. If you ran a slave deployment, redeploy as master against the prod DB. Future multi-region / HIPAA deployments will get a fresh service-auth design (mTLS / service JWTs). |
| 10 | **JWT carries `user_id` claim alongside `mobile_user_id`** (2026-04-29) | The user_conf + mobile_users merge unified the user table. JWTs minted post-merge include both `user_id` (canonical going forward) and `mobile_user_id` (back-compat) so in-flight tokens keep working. The two values are equal. | Prefer reading `user_id` from JWT payloads. The `mobile_user_id` claim will be removed after the refresh-token rollover window (~2026-05-29). |

### Changed Endpoints (same path, different behavior)

| Endpoint | Was (old) | Now (new) | Breaking? |
|----------|-----------|-----------|-----------|
| `POST /mobile/auth/signup` | 409 if email exists | 409 if verified email exists. Recycles unverified accounts with expired OTP | No |
| `POST /mobile/add-device` | Required `device_name`, `group_id`, `device_mac`. Device had to exist in DB (admin pre-registered). Error if already claimed. | `device_name`, `group_id`, and `device_mac` are all optional. Auto-creates device record. Returns `{status: "device_already_claimed"}` instead of error. Accepts `claim_token` alone (QR code flow — no MAC needed). MAC-like 12-hex-char tokens auto-resolve to MAC addresses. | **Partial** — check for `status` field in response |
| `POST /mobile/add-device` | Device auto-added to default group | Device starts in "Ungrouped Devices" system group | No |
| `POST /mobile/get-groups` | Returned `{data: {groups: [...]}}` | Returns `{data: {groups: [...], ungrouped_devices: []}}`. All devices are now in a group. `ungrouped_devices` is always empty (kept for backward compat). Each group has `is_default` field. | No — old `groups` key still works |
| `POST /mobile/get-groups` | No system group concept | "Ungrouped Devices" is a mandatory system group (`is_default: true`). Cannot be renamed, deleted, or have devices removed from it. Devices auto-move to it when removed from other groups. | No — additive change |
| `POST /mobile/get-groups` | Device/group fields: `id`, `name`, `mac_address` | Returns ONLY new names: `device_id`, `device_description`, `device_mac`, `group_id`, `group_name`. The legacy duplicate aliases were removed in the 2026-04-24 release. | **Yes** — clients still reading `id`/`name`/`mac_address` must switch to the new names |
| `POST /mobile/get-devices-with-alerts` | Response had `id`, `name`, `alertCount` | Returns BOTH old AND new field names (`device_id`, `device_description`, `alert_count`) | No |
| `POST /mobile/get-alerts-by-device` | Response had `id`, `current`, `threshold`, `alertStatus` | Returns BOTH old AND new field names (`alert_id`, `current_value`, `threshold_value`, `alert_status`) plus new `status`, `severity`, `acknowledged_at` | No |
| `POST /mobile/invite-user` | Required `invitee_email`. Returned `{invitation_id}` | `invitee_email` optional. Returns `{invitation_id, invite_token}`. Sends push notification | **Yes** — role must be `"manager"` not `"co-owner"` |
| `POST /mobile/delete-group` | Moved devices to default group | Devices move to "Ungrouped Devices" system group. Cannot delete the system group itself (400). | No |
| `POST /mobile/rename-group` | No restrictions | Cannot rename "Ungrouped Devices" system group (400) | **Partial** — handle 400 for default group |
| `POST /mobile/remove-device-from-group` | Device became ungrouped | Device moves to "Ungrouped Devices". Cannot remove from the system group itself (400). | **Partial** — handle 400 for default group |
| `POST /mobile/get-dashboard-stats` | Only counted owned devices | Counts owned + shared devices (via roles) | No — more accurate count |
| `POST /mobile/user-profile` | Returns caller's own profile only | Accepts optional `target_user_id` in body. Returns target's profile (only shared devices shown). Caller must share a device with target. | No — backward compatible |
| `POST /mobile/resolve-alert` | Any user with device access could resolve | Only owner, manager, caregiver can resolve. Viewers get 401. | **Partial** — viewers will now get errors |
| `POST /mobile/respond-invitation` | No notification sent | Sends push notification to inviter | No |
| `POST /mobile/get-vitals` | Returned ~2 data points per hour (full-range aggregation) | Returns ~240 data points per hour (auto-scaled). End timestamp clamped to server time. | No — more data, same format |
| `POST /mobile/update-device-user-role` | Owner or co-owner could change roles | Owner AND manager can change non-owner roles. Viewers and caregivers get 403. The owner's role cannot be changed via this endpoint (use `remove-device` to unclaim). | **Partial** — caregivers/viewers will now get 403 |
| `POST /mobile/user/delete-account` | Was a stub (501) | Now works — soft-deletes account | No |
| `POST /mobile/notifications/register` | Was a stub (501) | Now works — registers FCM token | No — was unused |
| `POST /mobile/notifications/preferences` | Was a stub (501) | Now works — toggles push/email | No — was unused |
| `POST /mobile/export-vitals` | Was a stub (501) | Now works — returns CSV or JSON file | No — was unused |
| `POST /mobile/get-device-users` | Returned `{id, name, role, status}` per user. Owner could be missing if they had no role row. | Returns `{id, name, email, role, status}`. Owner always included. Pending invitations resolve `id` from registered accounts. Expired pending invitations filtered out. | **Partial** — new `email` field added. `id` no longer null for registered pending users. |
| `POST /mobile/invite-user` | Allowed duplicate pending invitations for same email. Did not check if invitee already had access. | Rejects duplicate pending invitations (409). Rejects inviting users who already have access or are the device owner (409). Sets `invitee_user_id` when invitee has an account. | **Partial** — new 409 responses for duplicates |
| `POST /mobile/update-device-user-role` | Silently returned 200 when trying to change owner's role (no rows updated). Returned 200 for non-existent target users. | Returns 400 `"Cannot change the device owner's role"`. Returns 404 `"Target user not found on this device"`. | **Partial** — new error responses |
| `POST /mobile/get-device-users` (2026-04-29) | Pending invitations returned `id: null` when the invitee had no account, leaving the mobile app with no stable row key. | Each row now carries `invitation_id` alongside `id`. Pending rows have a non-null `invitation_id`; Active rows have `invitation_id: null`. | No — additive field |
| `POST /mobile/get-received-invitations` (renamed from `/mobile/get-invitations` on 2026-04-29) | Old path returned only Pending received invitations. Response per row had `invitation_id, invited_by, device, role, status, invited_at`. | New path returns every status (Pending + Active + Declined + Expired + Revoked) — symmetric with `get-sent-invitations`. Mobile app filters client-side. Response gains `note`, `expires_at`, `invitee_email`. Body stays `{}`. | **Yes** — endpoint path changed AND filter behavior changed. Update the URL to `/mobile/get-received-invitations` and filter by `status === 'Pending'` for the inbox view. The old `/mobile/get-invitations` path now returns 404. |
| `POST /mobile/accept-invite-link` (2026-04-29) | A targeted invite (where `invitee_email` was set) could only be accepted by an account whose registered email matched. | The email lock is now only enforced when the invitee already had an account at invite time (`invitee_user_id` set). Targeted invites to unregistered emails accept under any email — the recipient may have signed up under a different one. | No — strictly more permissive |
| `POST /mobile/remove-device-user`, `POST /mobile/update-device-user-role` (2026-04-30) | Body required exactly one of `target_user_id` or `target_email`. Pending invitations were targeted via the fuzzy `target_email` branch, which could match the wrong row when multiple Pending invitations existed for the same email + device. | Body now accepts a third option: `target_invitation_id`. Pass it for Pending rows (the value comes straight from `get-device-users`'s `invitation_id` field) for a deterministic single-row match. `target_user_id` is still preferred for Active users; `target_email` stays supported as a legacy fallback. The validator now requires **exactly one** of the three identifiers. | No — additive field |

### New Endpoints (not in previous version)

| Endpoint | Purpose | When to adopt |
|----------|---------|---------------|
| `POST /mobile/auth/google` | Google Sign-In via JWKS | When adding Google Sign-In to app |
| `POST /mobile/auth/apple` | Apple Sign-In via JWKS | When adding Apple Sign-In to app |
| `POST /mobile/auth/logout` | Revoke the current access + refresh tokens server-side | When adding sign-out (recommended — stops stolen tokens from being usable) |
| `POST /mobile/update-device` | Rename a device (owner/manager) | When adding device settings UI |
| `POST /mobile/remove-device` | Unclaim a device, release all access | When adding device removal from account |
| `POST /mobile/rename-group` | Rename a group | When adding group edit UI |
| `POST /mobile/request-device-access` | Request view access to another user's device | When adding device sharing discovery |
| `POST /mobile/acknowledge-alert` | Mark alert as "I'm handling it" (stops escalation) | When adding alert workflow |
| `POST /mobile/accept-invite-link` | Accept a shareable invite token | When adding invite link deep links |
| `POST /mobile/invite-user-to-group` | Invite user to ALL devices in a group | When adding group sharing |
| `POST /mobile/get-latest-vitals` | Latest reading for all devices in one call | For dashboard home screen |
| `POST /mobile/notifications/unregister` | Remove FCM token on logout | When implementing push |
| `POST /mobile/user-profile` | User details + associated devices. Optional `target_user_id` to view another user (must share a device). | For user profile and "view user details" in device user list |
| `POST /mobile/get-sent-invitations` (2026-04-29) | Outbox view — invitations the caller has sent. Returns all statuses (Pending/Active/Declined/Expired/Revoked) with `invite_token` for the copy-link UI action, plus `note`, `sent_at`, `expires_at`, `use_count`, `max_uses`. | When adding a "Sent" tab to the invitations page |
| `POST /mobile/resend-invitation` (2026-04-29) | Re-fires the invite email + push for a Pending invitation. Caller must be the original inviter. No DB state change (token, expiry, status preserved). | When adding a "Resend" button to the Sent invitations page |

### Removed Endpoints

| Old Endpoint | Replacement |
|-------------|-------------|
| `POST /mobile/auth/verify-firebase-token` | `POST /mobile/auth/google` or `POST /mobile/auth/apple` |

### Unchanged Endpoints (no modifications needed)

These endpoints work exactly as documented in the previous version:

`/mobile/auth/login`, `/mobile/auth/verify-otp`, `/mobile/auth/resend-otp`, `/mobile/auth/reset-password`, `/mobile/auth/update-password`, `/mobile/auth/refresh-token`, `/mobile/user/get-profile`, `/mobile/user/complete-profile`, `/mobile/user/update-profile`, `/mobile/user/update-status`, `/mobile/create-group`, `/mobile/add-device-to-group`, `/mobile/pair-device`, `/mobile/get-alert-thresholds`, `/mobile/update-alert-thresholds`, `/mobile/remove-device-user`

---

## Table of Contents

- [Migration Guide](#migration-guide-v2-api-changes)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
  - [Authentication](#authentication)
  - [Auth Endpoints](#auth-endpoints) (9 endpoints)
  - [Profile Endpoints](#profile-endpoints) (5 endpoints)
  - [Device Endpoints](#device-endpoints) (7 endpoints)
  - [Group Endpoints](#group-endpoints) (6 endpoints)
  - [Vitals Endpoints](#vitals-endpoints) (3 endpoints)
  - [Alert Endpoints](#alert-endpoints) (6 endpoints)
  - [Sharing & Permissions](#sharing--permissions) (9 endpoints)
  - [Notifications](#notifications) (4 endpoints)
  - [Admin Endpoints](#admin-endpoints)
  - [Backend Server Endpoints](#backend-server-endpoints)
- [Permission Roles](#permission-roles)
- [Response Format](#response-format)
- [Integration Notes](#integration-notes)
- [Deployment](#deployment)
- [Testing](#testing)

---

## Architecture

Single-master FastAPI backend backed by **PostgreSQL with TimescaleDB** (continuous aggregates for vitals rollups), Valkey for caching/rate-limiting/Celery broker, and Celery workers for async email + alert delivery. The legacy `IS_MASTER_SERVER` master/slave forwarding pattern was removed in 2026-04-29; future multi-region or HIPAA-compliant deployments will get a fresh service-auth design (mTLS / service JWTs).

Containers in production:
1. **dot_cloud** — FastAPI/Uvicorn serving the REST API (auto-generated OpenAPI at `/docs`)
2. **dot_celery_worker** — Celery worker for background tasks (email send, alert dispatch, push notifications)
3. **dot_celery_beat** — Celery beat for scheduled tasks
4. **dot_forward_ga** — MQTT subscriber that writes sensor data to TimescaleDB
5. **dot_postgres** — PostgreSQL + TimescaleDB
6. **dot_valkey** — Valkey (Redis-compatible) for cache, rate limit counters, Celery broker

---

## Quick Start

```bash
# Clone the API repo
git clone https://github.com/Cicada-Tech-Systems/homedots-api.git
cd homedots-api

# Install dependencies
pip install fastapi uvicorn sqlalchemy psycopg2-binary pydantic[email] pyjwt bcrypt slowapi python-dotenv celery valkey

# Configure (copy and fill in DB credentials)
cp .env.dev.example .env.dev

# SSH tunnel to PostgreSQL (or connect directly if on the same network)
ssh -L 5432:127.0.0.1:5432 ga.homedots.us

# Run tests (no external services needed)
python -m pytest tests/ -v

# Start development server
python run.py
# API docs at http://localhost:3425/docs
```

---

## Project Structure

```
app/interface/
├── fastapi_app/        # API endpoints (FastAPI routers)
│   ├── main.py             App factory, middleware, exception handlers
│   ├── routers/            Route handlers (one per domain)
│   ├── request_models.py   Pydantic request body models
│   ├── service_deps.py     Typed service dependency injection
│   ├── dependencies.py     Auth dependencies (JWT, backend token)
│   ├── validation.py       Legacy validation for backend routes
│   └── types.py            Shared TypedDicts
├── models/             # SQLAlchemy ORM models
├── services/           # Business logic (20+ services)
├── tasks/              # Celery async tasks
├── utils/              # Time, MAC, sanitization utilities
├── config.py           # Configuration from env vars
└── errors.py           # Error classes

web/                    # React dashboard (Vite + Tailwind)
v3/                     # Next-gen FastAPI rewrite
tests/                  # 202 tests (pytest, SQLite in-memory)
migrations/             # SQL migration scripts
docker_image/           # Docker Compose + infrastructure
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `JWT_SECRET` | Yes | - | Secret for JWT signing |
| `SQL_DRIVER` | Yes | `postgresql+psycopg2` | SQLAlchemy driver string |
| `SQL_HOST` | Yes | `postgres` | PostgreSQL host |
| `SQL_PORT` | Yes | `5432` | PostgreSQL port |
| `SQL_DATABASE` | Yes | `homedots` | Database name |
| `SQL_USER` | Yes | - | Database username |
| `SQL_PASSWORD` | Yes | - | Database password |
| `VALKEY_URL` | Yes | `redis://valkey:6379/0` | Valkey/Redis broker for cache + Celery |
| `EMAIL_SERVICE_URL` | No | - | Nodemailer HTTP endpoint |
| `FIREBASE_CREDENTIALS` | No | - | Firebase JSON (FCM push only) |
| `APP_BASE_URL` | No | `https://api.homedots.us` | Base URL for deep links |
| `GOOGLE_CLIENT_ID` | No | - | Google OAuth client ID |
| `APPLE_CLIENT_ID` | No | - | Apple OAuth client ID (bundle ID) |
| `SSL_VERIFY` | No | `true` | SSL verification for outbound requests |
| `LOG_LEVEL` | No | `WARNING` | Python logging level |
| `CORS_ORIGINS` | No | `https://homedots.us,...` | Allowed CORS origins |

`IS_MASTER_SERVER` and `MASTER_SERVER_URL` were removed in 2026-04-29. If you have them set in old config, drop them — they have no effect.

---

## API Reference

### Authentication

Two auth types:

| Type | Header | Used by |
|------|--------|---------|
| **JWT (Mobile)** | `Authorization: Bearer <jwt>` | Profile, notifications endpoints |
| **Backend Auth** | `Authorization: Bearer <backend_token>` | Device, vitals, alert, sharing endpoints |

JWT tokens are obtained via `/mobile/auth/login`, `/mobile/auth/verify-otp`, `/mobile/auth/google`, or `/mobile/auth/apple`. Access tokens expire in 1 hour. Use `/mobile/auth/refresh-token` to mint a fresh pair — the refresh endpoint also **revokes the old access token** (and the old refresh token), so callers should replace both after a refresh call.

**Token claims:** JWTs carry `jti` (unique id), `aud: "homedots-mobile"`, `iss: "homedots"`, `exp`, and `user_name` (resolved at mint time from the authoritative user record). Tokens that are revoked server-side — via `/mobile/auth/logout` or by the refresh-rotation step — are rejected even before expiry.

**`user_name` in request bodies:** Backend endpoints previously required `user_name` in the body. The server now derives the authoritative `user_name` from the JWT's `user_name` claim and only uses the body field as a legacy fallback when the claim is absent. Clients should keep sending it for compatibility, but **must not rely on being able to act as a different `user_name` than the token was minted for** — that path is closed.

**Rate limits (per IP):** mobile clients sit behind carrier NAT, so the limits favour real retry behaviour over abuse paranoia. Endpoints that send email stay tight; ones that don't are loose enough to absorb autofill/typo retries. Exceeding any limit returns `429 Too Many Requests`.

| Endpoint | Limit |
|---|---|
| `/mobile/auth/login` | **30 / min** |
| `/mobile/auth/verify-otp` | **15 / min** |
| `/mobile/auth/signup` | **10 / min** |
| `/mobile/auth/resend-otp` | 5 / min (sends email) |
| `/mobile/auth/reset-password` | 5 / min (sends email) |
| `/mobile/accept-invite-link` | 5 / min |

### Input Validation

| Field | Rule |
|-------|------|
| `email` | Valid format, auto-lowercased |
| `password` | Min 8 characters |
| `device_mac` | `XX:XX:XX:XX:XX:XX` format |
| `role` / `new_role` | `manager`, `viewer`, or `caregiver` |
| `otp_code` | Exactly 6 digits |
| `limit` | 1-200 (default 50) |
| `offset` | >= 0 (default 0) |

---

### Auth Endpoints

All `POST` to `/mobile/auth/*`. No authentication required except where noted (logout, update-password). Per-IP rate limits — see the table above; expect `429 Too Many Requests` past the cap.

**Email normalisation (2026-04-25):** every endpoint accepting `email` / `invitee_email` server-side lowercases + strips before any lookup or storage. Clients can keep sending whatever case the user typed; the backend canonicalises. Existing rows in the DB have been lower-cased to match.

<details>
<summary><b>POST /mobile/auth/signup</b> — Register a new account</summary>

**Body:** `{email: string, password: string}` (password min 8 chars)

| Response | Status | Body |
|----------|--------|------|
| Success | `201` | `{data: {mobile_user_id: 1}, message: "OTP sent"}` |
| Email already registered | `409` | `{message: "Email already registered"}` |
| Invalid email format | `400` | `{message: "Invalid email format"}` |
| Password too short | `400` | `{message: "Password must be at least 8 characters"}` |

If the email was previously registered but never verified (OTP expired), the account is recycled with a new password and fresh OTP.
</details>

<details>
<summary><b>POST /mobile/auth/login</b> — Login with email and password</summary>

**Body:** `{email: string, password: string}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {access_token, refresh_token, user: {mobile_user_id, email, name, status}}, message: "Login successful"}` |
| Wrong credentials | `401` | `{message: "Invalid email or password"}` |
| Account not verified | `401` | `{message: "Invalid email or password"}` |
| Account doesn't exist | `401` | `{message: "Invalid email or password"}` |
| OAuth-only account (no password) | `401` | `{message: "Invalid email or password"}` |

Note: All auth failures return the same message to prevent user enumeration.
</details>

<details>
<summary><b>POST /mobile/auth/verify-otp</b> — Verify OTP code</summary>

**Body:** `{email: string, otp_code: string (6 digits), otp_type: "signup" | "recovery"}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {access_token, refresh_token, user: {...}}, message: "Verified"}` |
| Wrong OTP | `400` | `{message: "Invalid or expired OTP"}` |
| OTP expired | `400` | `{message: "Invalid or expired OTP"}` |
| Wrong otp_type | `400` | `{message: "Invalid or expired OTP"}` |
| Too many attempts | `400` | `{message: "Too many failed attempts. Try again later."}` |
| User not found | `404` | `{message: "User not found"}` |

Locked out after 5 failed attempts for 15 minutes. `signup` type sets status to `verified`. `recovery` type enables password update for 15 minutes.
</details>

<details>
<summary><b>POST /mobile/auth/resend-otp</b> — Resend OTP code</summary>

**Body:** `{email: string, otp_type: "signup" | "recovery"}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "OTP resent"}` |
| Email not found | `200` | `{message: "OTP resent"}` |

Always returns 200 to prevent user enumeration. Resets any OTP lockout.
</details>

<details>
<summary><b>POST /mobile/auth/reset-password</b> — Request password reset</summary>

**Body:** `{email: string}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Recovery OTP sent"}` |
| Email not found | `200` | `{message: "Recovery OTP sent"}` |

Always returns 200. Sends a `recovery`-type OTP to the email.
</details>

<details>
<summary><b>POST /mobile/auth/update-password</b> — Set new password (requires JWT)</summary>

**Body:** `{password: string}` (min 8 chars)

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Password updated"}` |
| OTP not verified | `403` | `{message: "OTP verification required before password update"}` |
| OTP verification expired (>15 min) | `403` | `{message: "OTP verification required before password update"}` |
| User not found | `404` | `{message: "User not found"}` |

Must call `verify-otp` with `otp_type: "recovery"` first. Password update window expires after 15 minutes.
</details>

<details>
<summary><b>POST /mobile/auth/refresh-token</b> — Refresh JWT access token</summary>

**Body:** `{mobile_user_id: int, refresh_token: string}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {access_token, refresh_token}}` |
| Invalid token | `401` | `{message: "Invalid refresh token"}` |
| Expired token | `401` | `{message: "Invalid refresh token"}` |
| Already rotated | `401` | `{message: "Invalid refresh token"}` |
| Token revoked | `401` | `{message: "Invalid refresh token"}` |
| Rate limited | `429` | `{message: "Rate limit exceeded"}` |

On success the server rotates BOTH tokens: the old refresh token and the old access token it was paired with are both added to the revocation list. The client must replace **both** stored tokens with the returned pair — calls with the previous access token will start returning `401` after the refresh round-trip even though it hasn't hit its `exp` yet.
</details>

<details>
<summary><b>POST /mobile/auth/logout</b> — Revoke the current tokens server-side</summary>

**Body:** `{refresh_token?: string}` — optional. If supplied, both the refresh-token jti and the linked access-token jti are revoked. If omitted, only the access token from the `Authorization` header is revoked.

**Auth:** requires JWT (`Authorization: Bearer <access_token>`).

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Logged out"}` |
| Missing/expired access token | `401` | `{message: "Token has expired"}` or `{message: "Bearer token is missing"}` |

Revocation entries self-expire after the token's own `exp`, so the blocklist doesn't grow without bound.
</details>

<details>
<summary><b>POST /mobile/auth/google</b> — Sign in with Google</summary>

**Body:** `{id_token: string}` (Google ID token from Google SDK)

| Response | Status | Body |
|----------|--------|------|
| Success (existing user) | `200` | `{data: {access_token, refresh_token, user: {...}}, message: "Authenticated via Google"}` |
| Success (new account) | `200` | Same format. Account auto-created with `status: "verified"`. |
| Invalid token | `401` | `{message: "Invalid Google token"}` |
| Token expired | `401` | `{message: "Google token expired"}` |
| Email not verified | `401` | `{message: "Google email is not verified"}` |
</details>

<details>
<summary><b>POST /mobile/auth/apple</b> — Sign in with Apple</summary>

**Body:** `{id_token: string, email?: string, name?: string}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {access_token, refresh_token, user: {...}}, message: "Authenticated via Apple"}` |
| No email available | `400` | `{message: "Email is required. Apple may not include it after first login."}` |
| Email not verified | `401` | `{message: "Apple email is not verified"}` |
| Invalid token | `401` | `{message: "Invalid Apple token"}` |

Apple only sends `email` on first authorization. After that, the client must cache and send it in the request body. The `email_verified` claim is now validated — matches the Google sign-in behavior.
</details>

**Password reset flow:** `reset-password` → `verify-otp` (recovery) → `update-password`

---

### Profile Endpoints

All `POST` to `/mobile/user/*`. Require **JWT auth**.

<details>
<summary><b>POST /mobile/user/get-profile</b> — Get current user's profile</summary>

**Body:** _(none — user identified by JWT)_

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {id, email, name, age, gender, sleep_goals, wake_time, sleep_time, status, is_admin}}` |
| User not found | `404` | `{message: "User not found"}` |
| Missing/expired JWT | `401` | `{message: "Bearer token is missing"}` or `{message: "Token has expired"}` |
</details>

<details>
<summary><b>POST /mobile/user/complete-profile</b> — Complete profile after signup</summary>

**Body:** `{name, age, gender, sleep_goals, wake_time, sleep_time}` (all required)

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Profile completed"}` |
| Wrong status | `400` | `{message: "Cannot complete profile in current status 'not_verified'"}` |
| Missing fields | `400` | `{message: "Missing keys: name, age"}` |
</details>

<details>
<summary><b>POST /mobile/user/update-profile</b> — Update profile fields</summary>

**Body:** `{name?, age?, gender?, sleep_goals?, wake_time?, sleep_time?}` (send only fields to change)

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Profile updated"}` |

Only allowlisted fields are accepted. Any `email`, `password`, `status` fields in the body are silently ignored.
</details>

<details>
<summary><b>POST /mobile/user/update-status</b> — Update user status</summary>

**Body:** `{status: "not_verified" | "verified" | "profile_completed" | "tutorial_completed"}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Status updated"}` |
| Invalid status | `400` | `{message: "Invalid status. Must be one of: ..."}` |
</details>

<details>
<summary><b>POST /mobile/user/delete-account</b> — Delete account</summary>

**Body:** _(none)_

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Account deleted"}` |

Soft-deletes the account. User data is retained but inaccessible. Releases all owned devices.
</details>

---

### Device Endpoints

Require **backend auth** (`Authorization: Bearer <token>`) + `user_name` in body.

<details>
<summary><b>POST /mobile/add-device</b> — Claim a device by MAC address or QR claim token</summary>

Two modes of operation:

**Mode 1 — MAC address (manual entry):**
`{user_name, device_mac, device_name?, claim_token?, group_id?}`

**Mode 2 — QR code / claim token only (no MAC needed):**
`{user_name, claim_token, device_name?}`

| Response | Status | Body |
|----------|--------|------|
| Claimed successfully | `201` | `{data: {status: "claimed", device_id: 42}}` |
| Already claimed by another user | `200` | `{data: {status: "device_already_claimed", device_id: 42, message: "..."}}` |
| Wrong claim token | `400` | `{message: "Invalid claim token. Check the code on your device label."}` |
| Invalid MAC format | `400` | `{message: "Invalid MAC address format (expected XX:XX:XX:XX:XX:XX)"}` |
| Token not found | `404` | `{message: "No device found for this code. Check the QR code on your device."}` |
| User not found | `404` | `{message: "User not found"}` |

**QR code flow:** When only `claim_token` is sent (no `device_mac`), the backend looks up the device by its claim token. If the token is 12 hex characters (e.g. `AABBCCDDEEFF`), it's treated as a MAC address and auto-converted to `AA:BB:CC:DD:EE:FF`. If the device doesn't exist, it's auto-created.

**QR code URL format:** `https://api.homedots.us/claim/{claim_token}`

Each device has a unique 8-character claim token (e.g. `A7XK2M9B`) printed on its label as a QR code. The app scans the QR, extracts the token, and calls this endpoint.
</details>

<details>
<summary><b>POST /mobile/pair-device</b> — Pair a device using a pairing code</summary>

**Body:** `{user_name, pairing_code, device_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `201` | `{data: {device_id, device_mac}, message: "Device paired"}` |
| Invalid code | `404` | `{message: "Invalid or expired pairing code"}` |
| Already claimed | `400` | `{message: "Device is already claimed by another user"}` |
</details>

<details>
<summary><b>POST /mobile/update-device</b> — Rename a device</summary>

**Body:** `{user_name, device_mac, device_name?}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {device_id, device_mac, device_description}, message: "Device updated"}` |
| No access | `400` | `{message: "Only device owners and managers can edit device settings"}` |
| Device not found | `404` | `{message: "Device not found"}` |
</details>

<details>
<summary><b>POST /mobile/request-device-access</b> — Request view access to someone else's device</summary>

**Body:** `{user_name, device_mac, note?}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {invitation_id: 15}, message: "Access request sent to device owner"}` |
| Already have access | `400` | `{message: "You already have access to this device"}` |
| Own device | `400` | `{message: "You already own this device"}` |
| No owner | `400` | `{message: "Device has no owner to request access from"}` |
| Already pending | `400` | `{message: "Access request already pending for this device"}` |

Creates an invitation with `is_access_request=true` visible in the owner's pending invitations. Owner can accept or decline.
</details>

<details>
<summary><b>POST /mobile/remove-device</b> — Unclaim a device and release it</summary>

**Body:** `{user_name, device_mac, delete_vitals?: boolean}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {device_id, device_mac, vitals_deleted}, message: "Device removed"}` |
| Not owner | `400` | `{message: "Only the device owner can remove it"}` |
| Device not found | `404` | `{message: "Device not found"}` |

Releases ownership (`user_id` set to NULL), removes all `user_device_roles` (owner + shared users), removes from all groups. Device becomes claimable by anyone.

Set `delete_vitals: true` to also permanently erase all vital history from InfluxDB. Default is `false` (vitals kept). Call `/mobile/export-vitals` first if the user wants to save a copy.
</details>

<details>
<summary><b>POST /mobile/get-groups</b> — List all groups with devices</summary>

**Body:** `{user_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {groups: [{group_id, group_name, device_count, is_default, devices: [{device_id, device_description, device_mac, last_seen_raw, activity_status, role}]}], ungrouped_devices: []}}` |

Every device belongs to a group. The "Ungrouped Devices" group (`is_default: true`) is auto-created and cannot be renamed, deleted, or have devices removed from it. When a device is removed from any other group, it moves to this group automatically. `ungrouped_devices` is always `[]` (kept for backward compatibility).

Each group includes `is_default: true/false`. The default group sorts first. As of the 2026-04-24 release, only the **new** field names are returned — `device_id`, `device_description`, `device_mac` (for devices) and `group_id`, `group_name` (for groups). The legacy duplicates `id` / `name` / `mac_address` have been removed.
</details>

<details>
<summary><b>POST /mobile/get-latest-vitals</b> — Latest vitals for all devices</summary>

**Body:** `{user_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {vitals: {"AA:BB:CC:DD:EE:FF": {heartrate, respiratoryrate, systolic, diastolic, movement, quality, occupancy, time, on_bed, thresholds}, ...}}}` |
| No devices | `200` | `{data: {vitals: {}}}` |

Every accessible device appears in the response — including ones with no recent data (empty vitals, `on_bed: false`, thresholds still populated).

- `on_bed` — `true` when `occupancy >= 0.5`, `false` otherwise.
- When `on_bed` is `false`, the medical vitals (`heartrate`, `respiratoryrate`, `systolic`, `diastolic`) are returned as `null` so the dashboard card doesn't render garbage readings.
- `movement` is the **count** of data points with any movement recorded in the lookback window (default 1 hour), not the raw enum value. Higher number = more restless.
- `thresholds` merges the device's saved alert_setting on top of defaults, so a device with only `hr` customised still gets sensible defaults for `rr` / `bph` / `bpl` / `oc`. Shape matches `/mobile/get-alert-thresholds`: `{hr: {min, max}, rr: {min, max}, bph: {min, max}, bpl: {min, max}, oc: {on, off}}`.

One Postgres `last()` query against TimescaleDB regardless of device count. Safe to poll every 2-3 seconds.
</details>

---

### Group Endpoints

Require **backend auth** + `user_name` in body.

<details>
<summary><b>POST /mobile/create-group</b> — Create a device group</summary>

**Body:** `{user_name, group_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `201` | `{data: {group_id: 5}, message: "Group created"}` |
| Name empty | `400` | `{message: "group_name cannot be empty"}` |
| Name too long | `400` | `{message: "group_name too long (max 255 characters)"}` |
</details>

<details>
<summary><b>POST /mobile/rename-group</b> — Rename a group</summary>

**Body:** `{user_name, group_id, group_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Group renamed"}` |
| Not found | `404` | `{message: "Group not found"}` |
| Not owner | `404` | `{message: "Group not found"}` |
| Default group | `400` | `{message: "Cannot rename the default group"}` |
</details>

<details>
<summary><b>POST /mobile/delete-group</b> — Delete a group</summary>

**Body:** `{user_name, group_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Group deleted"}` |
| Not found | `404` | `{message: "Group not found"}` |
| Default group | `400` | `{message: "Cannot delete the default group"}` |

Devices in the deleted group are moved to "Ungrouped Devices" (not deleted).
</details>

<details>
<summary><b>POST /mobile/add-device-to-group</b> — Add a device to a group</summary>

**Body:** `{user_name, group_id, device_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Device added to group"}` |
| Not your group | `400` | `{message: "Group not found or not owned by user"}` |
| No device access | `400` | `{message: "No access to this device"}` |
</details>

<details>
<summary><b>POST /mobile/remove-device-from-group</b> — Remove a device from a group</summary>

**Body:** `{user_name, group_id, device_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Device removed from group"}` |
| Not your group | `400` | `{message: "Group not found or not owned by user"}` |
</details>

<details>
<summary><b>POST /mobile/invite-user-to-group</b> — Invite a user to all devices in a group</summary>

**Body:** `{user_name, group_id, invitee_email?, role}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {invited_count: 3, errors: []}, message: "Invited to 3 device(s)"}` |
| No devices | `400` | `{message: "Group has no devices"}` |
| Group not found | `404` | `{message: "Group not found"}` |

Errors for individual devices are returned in `errors` array (e.g. "already invited"). Successfully invited devices are not rolled back.
</details>

---

### Vitals Endpoints

Require **backend auth** + `user_name` in body.

<details>
<summary><b>POST /mobile/get-vitals</b> — Query vital signs time series</summary>

**Body:** `{user_name, device_mac, start_timestamp: int, end_timestamp: int}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {vitals: [{time, heartrate, respiratoryrate, systolic, diastolic, movement, quality, occupancy}, ...], thresholds: {hr, rr, bph, bpl, oc}}}` |
| No data | `400` | `{message: "Data not found"}` |
| No access | `401` | `{message: "Unauthorized"}` |

Timestamps are UNIX epoch (seconds). `end_timestamp` is clamped to server's current time. Auto-scales aggregation interval: 1h→15s, 24h→5min, 7d→30min, 30d→2h.

- Any bucket where `occupancy < 0.5` (off-bed) has its medical vitals (`heartrate`, `respiratoryrate`, `systolic`, `diastolic`) returned as `null`, so the chart shows gaps instead of garbage for time spent out of bed.
- `movement` is the **count** of data points with movement in each bucket, not the raw enum (averaging the enum gives meaningless values like `7`). Plot it as a histogram/bar chart, not a line.
- `thresholds` is static for the range — the device's saved `alert_setting` merged on top of defaults. Ships alongside the time series so the chart can render threshold bands without a second round trip.
- Smoothing is on by default: a trailing moving-average (N=10 buckets) over the gap-filled series, so the line stays continuous through quiet minutes.

Available measurements: `heartrate`, `respiratoryrate`, `systolic`, `diastolic`, `movement`, `quality`, `occupancy`.
</details>

<details>
<summary><b>POST /mobile/export-vitals</b> — Export vitals as file download</summary>

**Body:** `{user_name, device_mac, start_timestamp, end_timestamp, format?: "csv" | "json"}`

| Response | Status | Body |
|----------|--------|------|
| Success (CSV) | `200` | File download with `Content-Type: text/csv` |
| Success (JSON) | `200` | File download with `Content-Type: application/json` |
| No data | `200` | Empty CSV (headers only) or `{vitals: []}` |

Default format is `csv`. Response includes `Content-Disposition: attachment` header.
</details>

---

### Alert Endpoints

Require **backend auth** + `user_name` in body.

<details>
<summary><b>POST /mobile/get-devices-with-alerts</b> — List devices with alert info</summary>

**Body:** `{user_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {devices: [{device_id, device_mac, device_description, alert_count, role, ...}]}}` |

Each device includes both old field names (`id`, `name`, `alertCount`) and new names (`device_id`, `device_description`, `alert_count`).
</details>

<details>
<summary><b>POST /mobile/get-alerts-by-device</b> — Paginated alert history</summary>

**Body:** `{user_name, device_mac, limit?: int, offset?: int}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {alerts: [{alert_id, title, current_value, threshold_value, unit, alert_status, status, severity, resolved_at, acknowledged_at, created_at}, ...]}}` |
| No access | `401` | `{message: "No access to this device"}` |
| Device not found | `404` | `{message: "Device not found"}` |

`alert_status` is the legacy field (`"High"`, `"Medium"`, `"Low"`). `status` is the new lifecycle field (`"triggered"`, `"acknowledged"`, `"resolved"`). `severity` is the new severity (`"critical"`, `"warning"`, `"info"`).
</details>

<details>
<summary><b>POST /mobile/get-alert-thresholds</b> — Get alert threshold settings</summary>

**Body:** `{user_name, device_mac}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {thresholds: {heartrate: {min: 50, max: 120}, respiratoryrate: {min: 8, max: 25}, ...}}}` |
| No thresholds set | `200` | `{data: {thresholds: {}}}` |
| No access | `404` | `{message: "Device not found or no access"}` |
</details>

<details>
<summary><b>POST /mobile/update-alert-thresholds</b> — Set alert thresholds</summary>

**Body:** `{user_name, device_mac, thresholds: {heartrate: {min: 50, max: 120}, ...}}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Thresholds updated"}` |
| Not owner/manager | `403` | `{message: "Only device owners and managers can update thresholds"}` |
| Invalid JSON | `400` | `{message: "Thresholds must be a JSON object"}` |
</details>

<details>
<summary><b>POST /mobile/resolve-alert</b> — Resolve an alert</summary>

**Body:** `{user_name, alert_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Alert resolved"}` |
| Already resolved | `400` | `{message: "Alert is already resolved"}` |
| Viewer (no permission) | `401` | `{message: "Viewers cannot resolve alerts"}` |
| Alert not found | `404` | `{message: "Alert not found"}` |
</details>

<details>
<summary><b>POST /mobile/acknowledge-alert</b> — Acknowledge an alert</summary>

**Body:** `{user_name, alert_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Alert acknowledged"}` |
| Not triggered | `400` | `{message: "Cannot acknowledge alert in 'resolved' state"}` |
| No permission | `403` | `{message: "Only owner, manager, or caregiver can acknowledge alerts"}` |
| Alert not found | `404` | `{message: "Alert not found"}` |

Acknowledging stops escalation. The alert moves from `triggered` → `acknowledged`.
</details>

**Alert lifecycle:** `triggered` → `acknowledged` → `resolved`

**Escalation:** Unacknowledged alerts re-notify at 5min, 10min, 15min intervals.

**Deduplication:** Same device+measurement won't fire duplicate alerts while one is active.

**Severity routing:** `critical` → push + email, `warning` → push, `info` → in-app only.

---

### Sharing & Permissions

Require **backend auth** + `user_name` in body.

<details>
<summary><b>POST /mobile/get-device-users</b> — List users with access to a device</summary>

**Body:** `{user_name, device_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {users: [{id, invitation_id, name, email, role, status}, ...]}}` |
| No access | `401` | `{message: "No access to this device"}` |

Returns the device owner (always first, `role: "owner"`), active role holders (`status: "Active"`), and non-expired pending invitations (`status: "Pending"`). The `email` field is included for all users.

Per row:
- `id` — `user_id` of the account, or `null` for pending invitations to addresses that don't have an account yet.
- `invitation_id` — set on Pending rows so the mobile app has a stable row key when `id` is null. `null` on Active rows. Use this to target `/mobile/cancel-invitation`, `/mobile/update-invitation`, or `/mobile/resend-invitation` directly without going through email.

**Who can call:** Any user with access to the device (owner, manager, caregiver, viewer).
</details>

<details>
<summary><b>POST /mobile/invite-user</b> — Create a device invitation</summary>

**Body:** `{user_name, device_id, invitee_email?: string, role: "manager"|"caregiver"|"viewer", max_uses?: int}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {invitation_id: 12, invite_token: "abc123..."}, message: "Invitation created"}` |
| Not owner/manager | `403` | `{message: "Only device owners and managers can invite users"}` |
| Self-invite | `400` | `{message: "Cannot invite yourself"}` |
| Duplicate pending | `409` | `{message: "A pending invitation already exists for this email"}` |
| Already has access | `409` | `{message: "User already has access to this device"}` |
| Is device owner | `409` | `{message: "User is the device owner"}` |
| Invalid role | `400` | `{message: "Invalid role. Must be one of: caregiver, manager, viewer"}` |

Omit `invitee_email` to create a shareable link. Set `max_uses: null` for unlimited uses. Default is single-use. The `invite_token` in the response can be shared as `https://api.homedots.us/invite/<invite_token>`.

If `invitee_email` is provided: sends email + push notification to the invitee (if they have an account). When the invitee has a registered account, `invitee_user_id` is automatically resolved and stored.
</details>

<details>
<summary><b>POST /mobile/get-received-invitations</b> — Inbox view: invitations received by the current user</summary>

**Body:** `{}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {invitations: [{id, invitation_id, invited_by, device, role, status, note, invited_at, expires_at, invitee_email}, ...]}}` |

Returns every status (Pending + Active + Declined + Expired + Revoked) for invitations where the caller's `user_id` matches `invitee_user_id` OR their email matches `invitee_email`. Symmetric with `get-sent-invitations` — the mobile app filters client-side (typical users have few enough rows that returning everything beats two endpoints). `id` and `invitation_id` are the same value (kept duplicated for client backward compat). Past-expiry Pending rows are lazy-flipped to `Expired` on read.

**Renamed 2026-04-29:** previously `/mobile/get-invitations`. The old path now returns 404. Update mobile clients to use the new path.

**Use cases:**
- **Inbox view** — filter rows where `status === 'Pending'`.
- **Received history page** — render all rows; group/sort by status as desired.
</details>

<details>
<summary><b>POST /mobile/get-sent-invitations</b> — Outbox view: invitations the caller has sent</summary>

**Body:** `{}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {invitations: [{id, invitation_id, device_id, device, invitee_user_id, invitee_email, role, status, note, sent_at, expires_at, invite_token, use_count, max_uses}, ...]}}` |

Returns all invitations where `inviter_user_id == caller`, every status surfaced. The `invite_token` field lets the mobile app reconstruct the share link as `https://api.homedots.us/invite/<invite_token>` for the **Copy Link** action. Past-expiry Pending rows are lazy-flipped to `Expired` on read so the outbox stays consistent with the inbox state machine.

Pair with `/mobile/cancel-invitation` (Revoke), `/mobile/update-invitation` (change role), and `/mobile/resend-invitation` (re-fire email + push) to wire the per-row actions on the Sent invitations page.
</details>

<details>
<summary><b>POST /mobile/resend-invitation</b> — Re-fire email + push for a Pending invitation</summary>

**Body:** `{invitation_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Invitation resent"}` |
| Not the inviter | `401` | `{message: "Only the original inviter can resend this invitation"}` |
| Not Pending | `400` | `{message: "Cannot resend invitation in '<status>' state"}` |
| No email on invitation | `400` | `{message: "This invitation has no email to resend to"}` |
| Unknown invitation_id | `404` | `{message: "Invitation not found"}` |

No DB state change — `invite_token`, `expires_at`, and `status` are all preserved. The same email + push that fire on `/mobile/invite-user` are re-fired (push only delivers if the invitee has an account). Bouncing emails will bounce again; the caller-visible signal is the same as the original send.
</details>

<details>
<summary><b>POST /mobile/respond-invitation</b> — Accept or decline an invitation</summary>

**Body:** `{user_name, invitation_id, accept: bool}`

| Response | Status | Body |
|----------|--------|------|
| Accepted | `200` | `{message: "Invitation accepted"}` |
| Declined | `200` | `{message: "Invitation declined"}` |
| Not for you | `403` | `{message: "This invitation is not for you"}` |
| Already resolved | `404` | `{message: "Invitation not found or already resolved"}` |
| Expired | `404` | `{message: "Invitation has expired"}` |

Accepting creates a `UserDeviceRole` for the user. Sends push notification to the inviter.
</details>

<details>
<summary><b>POST /mobile/accept-invite-link</b> — Accept a shareable invite link</summary>

**Body:** `{user_name, invite_token: string}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {device_id: 42, role: "viewer"}, message: "Invitation accepted"}` |
| Invalid token | `404` | `{message: "Invalid invite link"}` |
| Expired | `404` | `{message: "This invite link has expired"}` |
| Usage limit reached | `400` | `{message: "This invite link has reached its usage limit"}` |
| Own invitation | `400` | `{message: "Cannot accept your own invitation"}` |
| Already have access | `400` | `{message: "You already have access to this device"}` |
| Not the targeted invitee (registered user only) | `401` | `{message: "This invitation is not for you"}` |

**Email-lock behavior** (changed 2026-04-29): when the invite was sent to a specific `invitee_email`, the link is locked to that email **only if the invitee already had an account at invitation time** (i.e. `invitee_user_id` was resolved server-side at invite creation). For invitations sent to addresses that didn't yet have an account, the recipient may sign up under any email of their choice and still claim the link — the email lock doesn't apply because we never had a real identity to anchor it to. Hijack-prevention is preserved for the case that matters (locking a known account), and onboarding is unblocked for the case where it doesn't (the email itself was the only identity proof).
</details>

<details>
<summary><b>POST /mobile/remove-device-user</b> — Remove a user's device access</summary>

**Body:** `{device_id, target_user_id?, target_email?, target_invitation_id?}` — provide **exactly one** target identifier:

- `target_user_id` — for accepted users (rows with `status: "Active"` in `get-device-users`).
- `target_invitation_id` — for Pending invitations. Deterministic single-row revoke — pass the `invitation_id` directly from `get-device-users`. Preferred over `target_email` when available.
- `target_email` — legacy fuzzy match for Pending invitations. Use only if the client doesn't have `invitation_id` surfaced yet.

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "User removed from device"}` |
| Wrong number of identifiers | `400` | `{message: "Provide exactly one of target_user_id, target_email, or target_invitation_id"}` |
| Not owner/manager | `401` | `{message: "Only device owners and managers can remove users"}` |
| Self-removal | `400` | `{message: "Cannot remove yourself from a device"}` |
| Target is device owner | `400` | `{message: "Cannot remove the device owner; unclaim the device instead"}` |
| Invitation not on this device | `404` | `{message: "Invitation not found on this device"}` |
| Target not on device & no pending invite | `404` | `{message: "User is not a member of this device"}` |

For the `target_invitation_id` branch the invitation lookup filters by **both** `invitation_id` AND `device_id`. A manager on device A cannot revoke an invitation on device B by guessing IDs — the response is the same 404 as a non-existent ID, no enumeration leak. The manage-access gate fires before any invitation lookup, so non-managers cannot probe IDs at all.

For the `target_user_id` branch: removes the `user_devices` row (if any) and revokes any matching pending invitations. Push notification fires only when an actual role row was removed — pending-invite-only revocations are silent (no app to notify).
</details>

<details>
<summary><b>POST /mobile/update-device-user-role</b> — Change a user's role on a device</summary>

**Body:** `{device_id, target_user_id?, target_email?, target_invitation_id?, new_role: "manager"|"caregiver"|"viewer"}` — same 3-way XOR target rule as `remove-device-user`. Use `target_invitation_id` for Pending rows so the role change lands on exactly that one invitation; `target_email` is the legacy fuzzy fallback.

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "User role updated"}` |
| Wrong number of identifiers | `400` | `{message: "Provide exactly one of target_user_id, target_email, or target_invitation_id"}` |
| Not owner/manager | `401` | `{message: "Only device owners and managers can change roles"}` |
| Target is owner | `400` | `{message: "Cannot change the device owner's role"}` |
| Invitation not on this device | `404` | `{message: "Invitation not found on this device"}` |
| Invitation not Pending | `400` | `{message: "Cannot change role on an invitation in '<status>' state"}` |
| Target not on device | `404` | `{message: "Target user not found on this device"}` |
| Invalid role | `400` | `{message: "Invalid role. Must be one of: caregiver, manager, viewer"}` |

Same security model as remove-device-user: invitation lookup is scoped to `device_id`, manage-access gate runs first.

Owner AND manager roles can change non-owner roles (manager-OK policy). Owner's own role is immutable here — use `remove-device` to unclaim instead. Push notification fires only on role rows; pending-invite role changes are silent.
</details>

<details>
<summary><b>POST /mobile/user-profile</b> — Get user details with associated devices</summary>

**Body:** `{user_name?: string, target_user_id?: int}` — validated by a Pydantic model. Passing a non-integer `target_user_id` now returns `422` with a structured validation error instead of the previous generic 500.

| Response | Status | Body |
|----------|--------|------|
| Own profile | `200` | `{data: {user_id, user_name, email, name, status, devices: [{device_id, device_mac, device_description, role}, ...]}}` |
| Other user's profile | `200` | Same format, but `devices` only includes devices shared between caller and target |
| User not found | `404` | `{message: "User not found"}` |
| No shared devices | `403` | `{message: "You do not share any devices with this user"}` |
| Bad `target_user_id` (not int) | `422` | Pydantic validation error envelope |

Without `target_user_id`, returns the authenticated user's profile with all their devices. Owned devices have `role: "owner"`, shared devices show the assigned role.

With `target_user_id`, returns that user's profile — but only devices that both caller and target share are included (privacy protection). Caller must share at least one device with the target.
</details>

---

### Notifications

<details>
<summary><b>POST /mobile/notifications/register</b> — Register FCM push token (JWT auth)</summary>

**Body:** `{device_token: string, device_info?: string}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Push token registered"}` |
| Missing token | `400` | `{message: "Missing keys: device_token"}` |
</details>

<details>
<summary><b>POST /mobile/notifications/unregister</b> — Remove FCM token (JWT auth)</summary>

**Body:** `{device_token: string}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Push token removed"}` |
</details>

<details>
<summary><b>POST /mobile/notifications/preferences</b> — Update notification preferences (JWT auth)</summary>

**Body:** `{push_enabled?: bool, email_enabled?: bool}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {push_enabled: true, email_enabled: false}}` |

Send only fields to change. Returns current state after update.
</details>

<details>
<summary><b>POST /mobile/notifications/preferences</b> — Get notification preferences (JWT auth)</summary>

**Body:** `{}` (empty body to read current state)

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {push_enabled: true, email_enabled: true}}` |

Send empty body to read. Send `{push_enabled, email_enabled}` to update. Defaults to both enabled if user hasn't set preferences.
</details>

**Push notifications sent for:** alert triggers, alert escalations, invitation received, invitation accepted/declined, access request received, role changed, user removed from device.

---

### Admin Endpoints

Require **admin JWT** (`is_admin=True` in token).

| Endpoint | Description |
|----------|-------------|
| `POST /api/v2/admin/tables/<table>/list` | List config table rows (mqtt_conf, influx_conf, forward_conf, ai_conf, tboard_conf) |
| `POST /api/v2/admin/tables/<table>/create` | Create config row |
| `POST /api/v2/admin/tables/<table>/update` | Update config row |
| `POST /api/v2/admin/tables/<table>/delete` | Delete config row |
| `POST /api/v2/admin/users/mobile` | List all mobile users |
| `POST /api/v2/admin/users/backend` | List all backend users |
| `POST /api/v2/admin/users/set-admin` | Toggle admin flag |
| `POST /api/v2/admin/users/delete` | Delete mobile user |
| `POST /api/v2/admin/devices/list-all` | List all devices |
| `POST /api/v2/admin/devices/update` | Update device config |
| `POST /api/v2/admin/devices/provision-tokens` | Bulk generate claim tokens for device MACs |
| `POST /api/v2/admin/devices/get-claim-token` | Look up claim token for a device |
| `POST /api/v2/admin/servers/list` | List forward + AI servers |
| `POST /api/v2/admin/stats` | Dashboard statistics |
| `POST /api/v2/admin/device-config/get` | Get device config by MAC |
| `POST /api/v2/admin/device-config/update` | Update device config |
| `POST /api/v2/admin/device-config/create` | Create device config |
| `POST /api/v2/admin/devices/register` | Bulk register devices |
| `POST /api/v2/admin/devices/activate-registration` | Activate pending registrations |
| `POST /api/v2/admin/beddot/configure` | Configure BedDot device |

### Backend Server Endpoints

Used by forward servers, AI servers, and devices to authenticate and fetch configs.

| Endpoint | Auth | Description |
|----------|------|-------------|
| `POST /api/v2/auth/verify-user` | Backend | Verify user token |
| `POST /api/v2/auth/verify-device` | Device | Verify device token |
| `POST /api/v2/auth/verify-forward` | Forward | Verify forward server |
| `POST /api/v2/auth/verify-ai` | AI | Verify AI server |
| `POST /api/v2/auth/verify-admin` | Admin | Verify admin token |
| `POST /api/v2/devices/vitals` | Device | Read device vitals |
| `POST /api/v2/devices/list-by-user` | Backend | Devices for user |
| `POST /api/v2/devices/list-by-forward` | Forward | Devices for forward server |
| `POST /api/v2/devices/list-by-ai` | AI | Devices for AI server |
| `POST /api/v2/devices/influx-conf` | Device | InfluxDB config for device |
| `POST /api/v2/devices/mqtt-conf-by-forward` | Forward | MQTT config for forward server |
| `POST /api/v2/devices/last-seen` | Forward | Record device heartbeat |
| `POST /api/v2/organizations/list` | Admin | List organizations |
| `POST /api/v2/organizations/create` | Admin | Create organization |
| `POST /api/v2/organizations/update` | Admin | Update organization |
| `POST /api/v2/grafana/info-by-org` | Admin | Grafana info for org |
| `POST /api/v2/grafana/refresh` | Admin | Refresh Grafana dashboards |

---

## Permission Roles

| | **Owner** | **Manager** | **Caregiver** | **Viewer** |
|---|---|---|---|---|
| View vitals & alerts | Yes | Yes | Yes | Yes |
| Resolve/acknowledge alerts | Yes | Yes | Yes | No |
| Update alert thresholds | Yes | Yes | No | No |
| Invite/remove users | Yes | Yes | No | No |
| Change user roles | Yes | No | No | No |
| Receive alert notifications | Yes | Optional | Yes | No |
| Export vitals data | Yes | Yes | Yes | No |
| Edit device name | Yes | Yes | No | No |
| Can be removed | No | Yes | Yes | Yes |

**Owner** is auto-assigned when a user adds/pairs a device. Other roles are assigned via invitations.

---

## Response Format

**Success:**
```json
{"data": {...}, "message": "Human-readable status"}
```

**Error:**
```json
{"message": "What went wrong", "request_id": "abc-123"}
```

Common HTTP codes: `400` (validation), `401` (unauthorized), `403` (forbidden), `404` (not found), `409` (conflict), `500` (server error).

---

## Integration Notes

**Auth flow for mobile apps:**
1. Signup: `/signup` → check email for OTP → `/verify-otp` → tokens returned
2. Login: `/login` → tokens returned
3. Google/Apple: `/auth/google` or `/auth/apple` → tokens returned (auto-creates account)
4. Store `access_token` in memory (not localStorage). Store `refresh_token` securely.
5. On 401: call `/refresh-token` → get new tokens. If that fails → re-login.

**Backend auth (device/group/alert/vitals endpoints):** These use a legacy backend token system. The JWT from login is automatically converted to a backend token by the `requires_backend_auth` decorator — the mobile app doesn't need to manage backend tokens.

**Role rename:** `co-owner` was renamed to `manager`. If the app sends `role: "co-owner"`, it will be rejected. Use `"manager"` instead.

**Devices are independent of groups.** `get-groups` returns both `groups` and `ungrouped_devices`. Devices start ungrouped when added.

---

## Deployment

```bash
# Build Docker image
docker build --no-cache -t dot_cloud:v2 .

# Start with Docker Compose
cd docker_image && docker compose up -d

# Run migrations on production DB
mysql -u dots -p beddot < migrations/004_schema_hardening.sql
mysql -u dots -p beddot < migrations/005_alert_system_overhaul.sql
```

Services: Backend (port 4000), MariaDB (internal), InfluxDB (port 4001), Grafana (port 4002).

---

## Testing

```bash
# All tests (165 passing)
python -m pytest tests/ --ignore=v3 -v

# Specific test file
python -m pytest tests/test_mobile_auth_service.py -v

# By keyword
python -m pytest tests/ --ignore=v3 -k "test_login" -v
```

Tests use SQLite in-memory — no external services needed.
