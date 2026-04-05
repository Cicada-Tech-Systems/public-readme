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

### Changed Endpoints (same path, different behavior)

| Endpoint | Was (old) | Now (new) | Breaking? |
|----------|-----------|-----------|-----------|
| `POST /mobile/auth/signup` | 409 if email exists | 409 if verified email exists. Recycles unverified accounts with expired OTP | No |
| `POST /mobile/add-device` | Required `device_name`, `group_id`. Device had to exist in DB (admin pre-registered). Error if already claimed. | `device_name` and `group_id` are optional. Auto-creates device record. Returns `{status: "device_already_claimed"}` instead of error. Accepts `claim_token`. | **Partial** — check for `status` field in response |
| `POST /mobile/add-device` | Device auto-added to default group | Device starts ungrouped | No |
| `POST /mobile/get-groups` | Returned `{data: {groups: [...]}}` | Returns `{data: {groups: [...], ungrouped_devices: [...]}}`. New `ungrouped_devices` array. | No — old `groups` key still works |
| `POST /mobile/get-groups` | Response had `is_default` on groups | `is_default` removed (no default groups) | No — just ignore missing field |
| `POST /mobile/get-groups` | Device fields: `id`, `name`, `mac_address` | Returns BOTH old (`id`, `name`, `mac_address`) AND new (`device_id`, `device_description`, `device_mac`) field names | No |
| `POST /mobile/get-devices-with-alerts` | Response had `id`, `name`, `alertCount` | Returns BOTH old AND new field names (`device_id`, `device_description`, `alert_count`) | No |
| `POST /mobile/get-alerts-by-device` | Response had `id`, `current`, `threshold`, `alertStatus` | Returns BOTH old AND new field names (`alert_id`, `current_value`, `threshold_value`, `alert_status`) plus new `status`, `severity`, `acknowledged_at` | No |
| `POST /mobile/invite-user` | Required `invitee_email`. Returned `{invitation_id}` | `invitee_email` optional. Returns `{invitation_id, invite_token}`. Sends push notification | **Yes** — role must be `"manager"` not `"co-owner"` |
| `POST /mobile/delete-group` | Moved devices to default group | Devices become ungrouped | No |
| `POST /mobile/resolve-alert` | Any user with device access could resolve | Only owner, manager, caregiver can resolve. Viewers get 401. | **Partial** — viewers will now get errors |
| `POST /mobile/respond-invitation` | No notification sent | Sends push notification to inviter | No |
| `POST /mobile/get-vitals` | Returned ~2 data points per hour (full-range aggregation) | Returns ~240 data points per hour (auto-scaled). End timestamp clamped to server time. | No — more data, same format |
| `POST /mobile/update-device-user-role` | Owner or co-owner could change roles | Only owner can change roles (managers cannot) | **Partial** — managers will now get 403 |
| `POST /mobile/user/delete-account` | Was a stub (501) | Now works — soft-deletes account | No |
| `POST /mobile/notifications/register` | Was a stub (501) | Now works — registers FCM token | No — was unused |
| `POST /mobile/notifications/preferences` | Was a stub (501) | Now works — toggles push/email | No — was unused |
| `POST /mobile/export-vitals` | Was a stub (501) | Now works — returns CSV or JSON file | No — was unused |

### New Endpoints (not in previous version)

| Endpoint | Purpose | When to adopt |
|----------|---------|---------------|
| `POST /mobile/auth/google` | Google Sign-In via JWKS | When adding Google Sign-In to app |
| `POST /mobile/auth/apple` | Apple Sign-In via JWKS | When adding Apple Sign-In to app |
| `POST /mobile/update-device` | Rename a device (owner/manager) | When adding device settings UI |
| `POST /mobile/remove-device` | Unclaim a device, release all access | When adding device removal from account |
| `POST /mobile/rename-group` | Rename a group | When adding group edit UI |
| `POST /mobile/request-device-access` | Request view access to another user's device | When adding device sharing discovery |
| `POST /mobile/acknowledge-alert` | Mark alert as "I'm handling it" (stops escalation) | When adding alert workflow |
| `POST /mobile/accept-invite-link` | Accept a shareable invite token | When adding invite link deep links |
| `POST /mobile/invite-user-to-group` | Invite user to ALL devices in a group | When adding group sharing |
| `POST /mobile/get-latest-vitals` | Latest reading for all devices in one call | For dashboard home screen |
| `POST /mobile/notifications/unregister` | Remove FCM token on logout | When implementing push |

### Removed Endpoints

| Old Endpoint | Replacement |
|-------------|-------------|
| `POST /mobile/auth/verify-firebase-token` | `POST /mobile/auth/google` or `POST /mobile/auth/apple` |

### Unchanged Endpoints (no modifications needed)

These endpoints work exactly as documented in the previous version:

`/mobile/auth/login`, `/mobile/auth/verify-otp`, `/mobile/auth/resend-otp`, `/mobile/auth/reset-password`, `/mobile/auth/update-password`, `/mobile/auth/refresh-token`, `/mobile/user/get-profile`, `/mobile/user/complete-profile`, `/mobile/user/update-profile`, `/mobile/user/update-status`, `/mobile/create-group`, `/mobile/add-device-to-group`, `/mobile/remove-device-from-group`, `/mobile/pair-device`, `/mobile/get-dashboard-stats`, `/mobile/get-alert-thresholds`, `/mobile/update-alert-thresholds`, `/mobile/get-device-users`, `/mobile/get-invitations`, `/mobile/remove-device-user`

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
  - [Sharing & Permissions](#sharing--permissions) (8 endpoints)
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

The backend runs in **dual mode** controlled by `IS_MASTER_SERVER`:

- **Master** (`IS_MASTER_SERVER=1`): Connects directly to MySQL, InfluxDB, Grafana.
- **Slave** (`IS_MASTER_SERVER=0`): Forwards API requests to the master server.

Three processes run in Docker via supervisord:
1. **http_api** — Flask/Gunicorn serving REST APIs + web dashboard
2. **socket_api** — Flask-SocketIO for WebSocket connections
3. **forward_ga** — MQTT subscriber that writes sensor data to InfluxDB

---

## Quick Start

```bash
# Clone and install
git clone <repo-url> && cd DotBackendLuo
pip install -r requirements-dev.txt

# Run tests (no external services needed)
python -m pytest tests/ --ignore=v3 -v

# Start server (requires .env with DB credentials)
cd app/interface/http_api && python http_api_server_main.py
```

---

## Project Structure

```
app/interface/
├── http_api/           # API endpoints (Flask blueprints)
│   ├── auth.py             Auth decorators (JWT, backend token)
│   ├── validators.py       Input validation (@validate decorator)
│   ├── mobile_auth_api.py
│   ├── mobile_devices_api.py
│   ├── mobile_alerts_api.py
│   ├── mobile_users_api.py
│   ├── mobile_vitals_api.py
│   ├── admin_dashboard_api.py
│   └── backend_*.py        Server-to-server endpoints
├── models/             # SQLAlchemy ORM models
├── services/           # Business logic (20+ services)
├── tasks/              # Celery async tasks
├── utils/              # Time, MAC, sanitization utilities
├── config.py           # Configuration from env vars
└── errors.py           # Error classes

web/                    # React dashboard (Vite + Tailwind)
v3/                     # Next-gen FastAPI rewrite
tests/                  # 165 tests (pytest, SQLite in-memory)
migrations/             # SQL migration scripts
docker_image/           # Docker Compose + infrastructure
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `IS_MASTER_SERVER` | Yes | `0` | `1` = direct DB, `0` = slave mode |
| `JWT_SECRET` | Yes | - | Secret for JWT signing |
| `SQL_HOST` | Master | `localhost` | MySQL/MariaDB host |
| `SQL_DATABASE` | Master | `beddot` | Database name |
| `SQL_USER` | Master | - | Database username |
| `SQL_PASSWORD` | Master | - | Database password |
| `MASTER_SERVER_URL` | Slave | - | URL of master server |
| `EMAIL_SERVICE_URL` | No | - | Nodemailer HTTP endpoint |
| `FIREBASE_CREDENTIALS` | No | - | Firebase JSON (FCM push only) |
| `APP_BASE_URL` | No | `https://api.homedots.us` | Base URL for deep links |
| `GOOGLE_CLIENT_ID` | No | - | Google OAuth client ID |
| `APPLE_CLIENT_ID` | No | - | Apple OAuth client ID (bundle ID) |
| `SSL_VERIFY` | No | `true` | SSL verification for outbound requests |
| `LOG_LEVEL` | No | `WARNING` | Python logging level |
| `CORS_ORIGINS` | No | `https://homedots.us,...` | Allowed CORS origins |

---

## API Reference

### Authentication

Two auth types:

| Type | Header | Used by |
|------|--------|---------|
| **JWT (Mobile)** | `Authorization: Bearer <jwt>` | Profile, notifications endpoints |
| **Backend Auth** | `Authorization: Bearer <backend_token>` + `user_name` in body | Device, vitals, alert, sharing endpoints |

JWT tokens are obtained via `/mobile/auth/login`, `/mobile/auth/verify-otp`, `/mobile/auth/google`, or `/mobile/auth/apple`. They expire in 1 hour. Use `/mobile/auth/refresh-token` to get a new one.

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

All `POST` to `/mobile/auth/*`. No authentication required. Rate-limited.

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

Old refresh token is invalidated on success (rotation). Store the new `refresh_token`.
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
| Invalid token | `401` | `{message: "Invalid Apple token"}` |

Apple only sends `email` on first authorization. After that, the client must cache and send it in the request body.
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
<summary><b>POST /mobile/add-device</b> — Claim a device by MAC address</summary>

**Body:** `{user_name, device_mac, device_name?, claim_token?, group_id?}`

| Response | Status | Body |
|----------|--------|------|
| Claimed successfully | `201` | `{data: {status: "claimed", device_id: 42}}` |
| Already claimed by another user | `200` | `{data: {status: "device_already_claimed", device_id: 42, message: "This device is registered to another user. You can request view access from the owner."}}` |
| Wrong claim token | `400` | `{message: "Invalid claim token. Check the code on your device label."}` |
| Invalid MAC format | `400` | `{message: "Invalid MAC address format (expected XX:XX:XX:XX:XX:XX)"}` |
| User not found | `404` | `{message: "User not found"}` |

If the device doesn't exist in the database, it's auto-created. `claim_token` is optional for testing but should be enforced in production. If device is already claimed, use `/mobile/request-device-access` to request view permissions.
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

**Body:** `{user_name, device_mac}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {device_id, device_mac}, message: "Device removed"}` |
| Not owner | `400` | `{message: "Only the device owner can remove it"}` |
| Device not found | `404` | `{message: "Device not found"}` |

Releases ownership (`user_id` set to NULL), removes all `user_device_roles` (owner + shared users), removes from all groups. Device becomes claimable by anyone.
</details>

<details>
<summary><b>POST /mobile/get-groups</b> — List all groups and ungrouped devices</summary>

**Body:** `{user_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {groups: [{group_id, group_name, device_count, devices: [{device_id, device_description, device_mac, last_seen_raw, activity_status, role}]}], ungrouped_devices: [...]}}` |

Devices not in any group appear in `ungrouped_devices`. Each device includes both old field names (`id`, `name`, `mac_address`) and new field names (`device_id`, `device_description`, `device_mac`) for backward compatibility.
</details>

<details>
<summary><b>POST /mobile/get-latest-vitals</b> — Latest vitals for all devices</summary>

**Body:** `{user_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {vitals: {"AA:BB:CC:DD:EE:FF": {time, heartrate, respiratoryrate, systolic, diastolic, movement, quality}, ...}}}` |
| No devices | `200` | `{data: {vitals: {}}}` |

Single InfluxDB query using `LAST()` aggregate. Safe to poll every 2-3 seconds.
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
</details>

<details>
<summary><b>POST /mobile/delete-group</b> — Delete a group</summary>

**Body:** `{user_name, group_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Group deleted"}` |
| Not found | `404` | `{message: "Group not found"}` |

Devices in the group become ungrouped (not deleted).
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
| Success | `200` | `{data: {vitals: [{time, heartrate, respiratoryrate, systolic, diastolic, movement, quality}, ...]}}` |
| No InfluxDB config | `400` | `{message: "Influx configuration not found for this device"}` |
| No data | `400` | `{message: "Data not found"}` |
| No access | `401` | `{message: "Unauthorized"}` |

Timestamps are UNIX epoch (seconds). `end_timestamp` is clamped to server's current time. Auto-scales aggregation interval: 1h→15s, 24h→5min, 7d→30min, 30d→2h.

Available measurements: `heartrate`, `respiratoryrate`, `systolic`, `diastolic`, `movement`, `quality`.
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
| Success | `200` | `{data: {users: [{id, name, role, status}, ...]}}` |
| No access | `401` | `{message: "No access to this device"}` |

Returns both active role holders (`status: "Active"`) and pending invitations (`status: "Pending"`).
</details>

<details>
<summary><b>POST /mobile/invite-user</b> — Create a device invitation</summary>

**Body:** `{user_name, device_id, invitee_email?: string, role: "manager"|"caregiver"|"viewer", max_uses?: int}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {invitation_id: 12, invite_token: "abc123..."}, message: "Invitation created"}` |
| Not owner/manager | `403` | `{message: "Only device owners and managers can invite users"}` |
| Self-invite | `400` | `{message: "Cannot invite yourself"}` |
| Duplicate pending | `400` | `{message: "Invitation already pending for this user and device"}` |
| Invalid role | `400` | `{message: "Invalid role. Must be one of: caregiver, manager, viewer"}` |

Omit `invitee_email` to create a shareable link. Set `max_uses: null` for unlimited uses. Default is single-use. The `invite_token` in the response can be shared as `https://api.homedots.us/invite/<invite_token>`.

If `invitee_email` is provided: sends email + push notification to the invitee (if they have an account).
</details>

<details>
<summary><b>POST /mobile/get-invitations</b> — List pending invitations for current user</summary>

**Body:** `{user_name}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{data: {invitations: [{invitation_id, invited_by, device, role, status, invited_at}, ...]}}` |

Shows invitations where the user's email matches `invitee_email`.
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
</details>

<details>
<summary><b>POST /mobile/remove-device-user</b> — Remove a user's device access</summary>

**Body:** `{user_name, device_id, target_user_id}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "User removed from device"}` |
| Not owner/manager | `403` | `{message: "Only device owners and managers can remove users"}` |
| Self-removal | `400` | `{message: "Cannot remove yourself from a device"}` |
| Target is owner | (no rows deleted) | Returns success with `false` — owner role is protected. |

Sends push notification to the removed user.
</details>

<details>
<summary><b>POST /mobile/update-device-user-role</b> — Change a user's role on a device</summary>

**Body:** `{user_name, device_id, target_user_id, new_role: "manager"|"caregiver"|"viewer"}`

| Response | Status | Body |
|----------|--------|------|
| Success | `200` | `{message: "Role updated"}` |
| Not owner | `403` | `{message: "Only device owners can change roles"}` |
| Invalid role | `400` | `{message: "Invalid role. Must be one of: caregiver, manager, viewer"}` |

Only the device owner can change roles (managers cannot). Owner role cannot be changed. Sends push notification to the affected user.
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
