# HomeDots API Backend

Healthcare IoT backend for BedDot sensor devices. Provides device management, vital signs data, user authentication, and mobile app APIs.

---

## Table of Contents

- [Deployment](#deployment)
  - [Build Docker Images](#build-docker-images)
  - [Deploy with Docker Compose](#deploy-with-docker-compose)
  - [Caddy Reverse Proxy](#caddy-reverse-proxy)
- [API Reference](#api-reference)
  - [Response Format](#response-format)
  - [Authentication Types](#authentication-types)
  - [Input Validation](#input-validation)
  - [Mobile Auth API](#mobile-auth-api) (7 endpoints)
  - [Mobile Devices API](#mobile-devices-api) (8 endpoints)
  - [Mobile Alerts API](#mobile-alerts-api) (4 endpoints)
  - [Mobile Users API](#mobile-users-api) (15 endpoints)
  - [Mobile Vitals API](#mobile-vitals-api) (2 endpoints)
  - [Legacy Device API](#legacy-device-api) (33 endpoints)
- [Integration Notes](#integration-notes)
- [Legacy Install (Non-Docker)](#legacy-install-non-docker)
- [Machine Code Conversion](#machine-code-conversion)

---

## Deployment

### Build Docker Images

```bash
# Build all 3 images
./build.sh

# Build and save as tar files (for offline transfer)
./build.sh --save
```

Images built:
| Image | Description |
|-------|-------------|
| `dot_cloud:v2` | Backend application |
| `dot_influx:wave` | InfluxDB with pre-configured databases and users |
| `dot_grafana:wave` | Grafana with custom dashboards |

### Deploy with Docker Compose

```bash
cd docker_image
chmod +x run_dot.sh
sudo ./run_dot.sh
```

This loads/pulls images, creates persistent directories at `/.dots/`, and starts 4 containers:

| Service | Container | Host Port | Internal Port |
|---------|-----------|-----------|---------------|
| Backend | dot_cloud | 4000 | 3425 |
| InfluxDB | influx | 4001 | 8086 |
| Grafana | grafana | 4002 | 3000 |
| MariaDB | dot_mariadb | none (internal) | 3306 |

All ports bind to `127.0.0.1` — only accessible via Caddy reverse proxy.

**First-time setup:**

1. Edit MariaDB credentials: `sudo nano /.dots/backend/.env.mariadb`
2. Update backend config in `/.dots/backend/.env`:
   - `SQL_HOST=dot_mariadb`
   - `INFLUXDB_IP=http://influx`
3. Restart: `docker restart dot_cloud`

### Caddy Reverse Proxy

Install Caddy on the host and copy `docker_image/Caddyfile` to `/etc/caddy/Caddyfile`. Edit the domain name, then restart Caddy. Caddy handles TLS automatically via Let's Encrypt.

---

## API Reference

### Response Format

All endpoints return JSON. The shape depends on success or failure:

**Success** (HTTP 200):
```json
{
  "data": { ... },
  "message": "Human-readable status"
}
```

**Error** (HTTP 4xx/5xx):
```json
{
  "message": "What went wrong"
}
```

Common error codes: `400` (bad request / validation), `401` (unauthorized / expired token), `403` (forbidden), `404` (not found), `409` (conflict / duplicate), `500` (server error).

---

### Authentication Types

| Type | Header | Used by | Description |
|------|--------|---------|-------------|
| JWT (Mobile) | `Authorization: Bearer <jwt_token>` | Profile endpoints | For mobile app users. Obtained via `/mobile/auth/login` or `/mobile/auth/verify-otp`. Expires in 1 hour. |
| Backend Auth | `Authorization: Bearer <backend_token>` | Device, alert, user-mgmt, vitals endpoints | Requires `user_name` in JSON body. Token is from the `user_conf` table. |

> JWT payload contains: `mobile_user_id`, `email`, `user_id`, `exp`. Profile endpoints use the `email` from the JWT automatically -- no need to send it in the body.

---

### Input Validation

The following rules are enforced server-side on all endpoints that accept these fields:

| Field | Rule |
|-------|------|
| `email` | Must be a valid email format. Automatically lowercased and trimmed. |
| `password` | Minimum 8 characters. |
| `device_mac` | Must match `XX:XX:XX:XX:XX:XX` format (hex, colon or dash separated). |
| `role` / `new_role` | Must be one of: `co-owner`, `viewer`, `caregiver`. |
| `status` | Must be one of: `not_verified`, `verified`, `profile_completed`, `tutorial_completed`. |
| `otp_type` | Must be `signup` or `recovery`. |
| `limit` | Clamped to 1-200. Defaults to 50. |
| `offset` | Clamped to >= 0. Defaults to 0. |

---

### Mobile Auth API

All endpoints under `/mobile/auth/`. No authentication required (public). Rate-limited per IP.

<details>
<summary><code>POST</code> <code><b>/mobile/auth/signup</b></code> <code>Register a new user account</code></summary>

Rate limit: 5/min

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `email` | required | string | User email address |
> | `password` | required | string | Min 8 characters |

##### Success Response (200)

```json
{
  "data": { "mobile_user_id": 1 },
  "message": "OTP sent"
}
```

> An OTP (6-digit code, type `signup`) is generated and must be verified before the user can log in.

##### Error Responses

> | http code | message |
> |-----------|---------|
> | `400` | Invalid email format / Password too short |
> | `409` | Email already registered |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/auth/login</b></code> <code>Login and receive JWT tokens</code></summary>

Rate limit: 10/min

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `email` | required | string | User email address |
> | `password` | required | string | User password |

##### Success Response (200)

```json
{
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "a1b2c3...",
    "user": {
      "mobile_user_id": 1,
      "email": "user@example.com",
      "name": "John",
      "status": "profile_completed"
    }
  },
  "message": "Login successful"
}
```

##### Error Responses

> | http code | message |
> |-----------|---------|
> | `401` | Invalid email or password |
> | `400` | Email not verified. Please verify your OTP first. |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/auth/verify-otp</b></code> <code>Verify OTP code sent to email</code></summary>

Rate limit: 5/min. Locked out after 5 failed attempts (15-minute cooldown).

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `email` | required | string | User email address |
> | `otp_code` | required | string | 6-digit OTP from email |
> | `otp_type` | required | string | `signup` or `recovery` |

##### Success Response (200)

```json
{
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "a1b2c3...",
    "user": {
      "mobile_user_id": 1,
      "email": "user@example.com",
      "name": null,
      "status": "verified"
    }
  },
  "message": "Verified"
}
```

> When `otp_type` is `signup`, user status changes from `not_verified` to `verified`.
> When `otp_type` is `recovery`, a one-time flag is set that permits `/mobile/auth/update-password`.

##### Error Responses

> | http code | message |
> |-----------|---------|
> | `400` | Invalid or expired OTP |
> | `400` | Too many failed attempts. Try again later. |
> | `404` | User not found |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/auth/resend-otp</b></code> <code>Resend OTP code</code></summary>

Rate limit: 3/min

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `email` | required | string | User email address |
> | `otp_type` | required | string | `signup` or `recovery` |

##### Success Response (200)

```json
{ "message": "OTP resent" }
```

> Always returns 200 even if email is not registered (prevents user enumeration). Resets any OTP lockout.

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/auth/reset-password</b></code> <code>Request password reset (sends recovery OTP)</code></summary>

Rate limit: 3/min

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `email` | required | string | User email address |

##### Success Response (200)

```json
{ "message": "Recovery OTP sent" }
```

> Always returns 200 even if email is not registered (prevents user enumeration). Sends a `recovery`-type OTP.

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/auth/update-password</b></code> <code>Set new password after OTP verification</code></summary>

Rate limit: 5/min

> **Prerequisite:** Must call `/mobile/auth/verify-otp` with `otp_type: "recovery"` first. The password update is only allowed once per recovery OTP verification.

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `email` | required | string | User email address |
> | `password` | required | string | New password (min 8 characters) |

##### Success Response (200)

```json
{ "message": "Password updated" }
```

> After a password change, the user's refresh token is invalidated. They must log in again.

##### Error Responses

> | http code | message |
> |-----------|---------|
> | `403` | OTP verification required before password update |
> | `404` | User not found |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/auth/refresh-token</b></code> <code>Refresh JWT access token</code></summary>

Rate limit: 10/min

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `mobile_user_id` | required | int | The user's mobile_user_id |
> | `refresh_token` | required | string | Current refresh token |

##### Success Response (200)

```json
{
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "new_token..."
  }
}
```

> The old refresh token is invalidated and a new one is returned (token rotation). Store the new `refresh_token` for the next refresh.

##### Error Responses

> | http code | message |
> |-----------|---------|
> | `401` | Invalid refresh token (wrong, expired, or already rotated) |

</details>

---

### Mobile Devices API

All endpoints require **backend authentication** (`Authorization: Bearer <backend_token>`) and `user_name` in the JSON body.

<details>
<summary><code>POST</code> <code><b>/mobile/get-groups</b></code> <code>Get all device groups for the user</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username (from user_conf) |

##### Success Response (200)

```json
{
  "data": {
    "groups": [
      {
        "id": 1,
        "name": "Bedroom",
        "device_count": 2,
        "devices": [
          {
            "id": 10,
            "name": "Mom's BedDot",
            "mac_address": "AA:BB:CC:DD:EE:01",
            "activity_status": "active",
            "occupied_status": "occupied",
            "role": "owner"
          }
        ]
      }
    ]
  }
}
```

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/create-group</b></code> <code>Create a new device group</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `group_name` | required | string | Name for the new group |

##### Success Response (200)

```json
{
  "data": { "group_id": 5 },
  "message": "Group created"
}
```

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/add-device-to-group</b></code> <code>Add a device to a group</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `group_id` | required | int | Target group ID |
> | `device_id` | required | int | Device ID to add |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/remove-device-from-group</b></code> <code>Remove a device from a group</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `group_id` | required | int | Group ID |
> | `device_id` | required | int | Device ID to remove |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/delete-group</b></code> <code>Delete a device group</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `group_id` | required | int | Group ID to delete |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/pair-device</b></code> <code>Pair a device using a pairing code</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `pairing_code` | required | string | Code displayed on device |
> | `device_name` | required | string | Friendly name for the device |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/add-device</b></code> <code>Manually add a device by MAC address</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `device_mac` | required | string | Device MAC address (`XX:XX:XX:XX:XX:XX`) |
> | `device_name` | required | string | Friendly name for the device |
> | `group_id` | required | int | Group to place device in |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/get-dashboard-stats</b></code> <code>Get dashboard statistics for the user</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |

Returns device counts and summary data.

</details>

---

### Mobile Alerts API

All endpoints require **backend authentication** (`Authorization: Bearer <backend_token>`) and `user_name` in the JSON body.

<details>
<summary><code>POST</code> <code><b>/mobile/get-devices-with-alerts</b></code> <code>Get all devices with alert info</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |

##### Success Response (200)

```json
{
  "data": {
    "devices": [
      {
        "id": 10,
        "created_at": "2025-01-15 10:30:00",
        "updated_at": "2025-06-01 08:00:00",
        "name": "Mom's BedDot",
        "heart_rate": null,
        "respiration_rate": null,
        "systolic": null,
        "diastolic": null,
        "role": "owner",
        "alertCount": 3
      }
    ]
  }
}
```

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/get-alerts-by-device</b></code> <code>Get paginated alerts for a device</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `device_mac` | required | string | Device MAC address (`XX:XX:XX:XX:XX:XX`) |
> | `limit` | optional | int | Number of alerts (1-200, default 50) |
> | `offset` | optional | int | Pagination offset (default 0) |

##### Success Response (200)

```json
{
  "data": {
    "alerts": [
      {
        "id": 42,
        "created_at": "2025-06-01 02:30:00",
        "updated_at": "2025-06-01 02:30:00",
        "title": "High Heart Rate",
        "current": 110,
        "threshold": 100,
        "unit": "bpm",
        "alertStatus": "active",
        "issueTime": "2025-06-01 02:28:00"
      }
    ]
  }
}
```

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/get-alert-thresholds</b></code> <code>Get alert threshold settings for a device</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `device_mac` | required | string | Device MAC address |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/update-alert-thresholds</b></code> <code>Update alert thresholds for a device</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `device_mac` | required | string | Device MAC address |
> | `thresholds` | required | object | Threshold values for vital signs |

</details>

---

### Mobile Users API

#### Profile Endpoints

Require **JWT authentication** (`Authorization: Bearer <jwt_token>`). The user's email is read from the JWT -- do not send `email` in the body.

<details>
<summary><code>POST</code> <code><b>/mobile/user/get-profile</b></code> <code>Get user profile data</code></summary>

No body parameters required. User is identified by JWT.

##### Success Response (200)

```json
{
  "data": {
    "id": 1,
    "email": "user@example.com",
    "name": "John Doe",
    "age": "35-44",
    "gender": "male",
    "sleep_goals": "Improve deep sleep",
    "wake_time": "07:00",
    "sleep_time": "22:00",
    "status": "profile_completed"
  }
}
```

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/user/complete-profile</b></code> <code>Complete user profile after signup</code></summary>

> Sets status to `profile_completed` automatically.

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `name` | required | string | Full display name |
> | `age` | required | string | Age range (e.g. `"35-44"`) |
> | `gender` | required | string | Gender |
> | `sleep_goals` | required | string | Sleep goals description |
> | `wake_time` | required | string | Wake time (`HH:MM` format) |
> | `sleep_time` | required | string | Sleep time (`HH:MM` format) |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/user/update-profile</b></code> <code>Update profile fields</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `name` | optional | string | Full display name |
> | `age` | optional | string | Age range |
> | `gender` | optional | string | Gender |
> | `sleep_goals` | optional | string | Sleep goals description |
> | `wake_time` | optional | string | Wake time (`HH:MM`) |
> | `sleep_time` | optional | string | Sleep time (`HH:MM`) |

> Only send the fields you want to change. Any `email` field in the body is ignored.

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/user/update-status</b></code> <code>Update user status</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `status` | required | string | One of: `not_verified`, `verified`, `profile_completed`, `tutorial_completed` |

</details>

#### Device User Management

Require **backend authentication** (`Authorization: Bearer <backend_token>`) and `user_name` in the JSON body. Only device owners and co-owners can invite, remove, or change roles.

<details>
<summary><code>POST</code> <code><b>/mobile/get-device-users</b></code> <code>Get all users with access to a device</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `device_id` | required | int | Device ID |

##### Success Response (200)

```json
{
  "data": {
    "users": [
      { "id": 1, "name": "owner_user", "role": "owner", "status": "Active" },
      { "id": null, "name": "invited@example.com", "role": "viewer", "status": "Pending" }
    ]
  }
}
```

> Returns both active role holders and pending invitations for the device.

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/invite-user</b></code> <code>Invite a user to share device access</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username (must be owner or co-owner) |
> | `device_id` | required | int | Device ID |
> | `invitee_email` | required | string | Email of user to invite |
> | `role` | required | string | `co-owner`, `viewer`, or `caregiver` |

##### Success Response (200)

```json
{
  "data": { "invitation_id": 12 },
  "message": "Invitation sent"
}
```

##### Error Responses

> | http code | message |
> |-----------|---------|
> | `403` | Only device owners/co-owners can invite users |
> | `400` | Invitation already pending for this user and device |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/get-invitations</b></code> <code>Get invitations for current user</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |

##### Success Response (200)

```json
{
  "data": {
    "invitations": [
      {
        "id": 12,
        "invited_by": "owner_user",
        "device": "Mom's BedDot",
        "role": "viewer",
        "status": "Pending",
        "invited_at": "2025-06-01 10:00:00"
      }
    ]
  }
}
```

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/respond-invitation</b></code> <code>Accept or decline an invitation</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username (must be the invitation recipient) |
> | `invitation_id` | required | int | Invitation ID |
> | `accept` | required | boolean | `true` to accept, `false` to decline |

> The server verifies the responder is the intended recipient (by matching `user_name` to the invitee's email via their linked mobile account). Accepting creates a role on the device; declining marks the invitation as declined.

##### Error Responses

> | http code | message |
> |-----------|---------|
> | `403` | This invitation is not for you |
> | `404` | Invitation not found or already resolved |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/remove-device-user</b></code> <code>Remove a user's access to a device</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username (must be owner or co-owner) |
> | `device_id` | required | int | Device ID |
> | `target_user_id` | required | int | User ID to remove |

> Cannot remove yourself. Cannot remove the device owner.

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/update-device-user-role</b></code> <code>Change a user's role on a device</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username (must be owner or co-owner) |
> | `device_id` | required | int | Device ID |
> | `target_user_id` | required | int | Target user ID |
> | `new_role` | required | string | `co-owner`, `viewer`, or `caregiver` |

> Cannot change the owner's role.

</details>

#### Future Endpoints (return 501)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/mobile/user/delete-account` | Delete user account |
| POST | `/mobile/notifications/register` | Register push notification token |
| POST | `/mobile/notifications/preferences` | Set notification preferences |
| POST | `/mobile/sleep/analysis` | Sleep data analysis |
| POST | `/mobile/sleep/report` | Generate sleep report |

---

### Mobile Vitals API

Require **backend authentication** (`Authorization: Bearer <backend_token>`) and `user_name` in the JSON body.

<details>
<summary><code>POST</code> <code><b>/mobile/get-vitals</b></code> <code>Get vital signs data for a device</code></summary>

##### Body Parameters

> | name | type | data type | description |
> |------|------|-----------|-------------|
> | `user_name` | required | string | Username |
> | `device_mac` | required | string | Device MAC address (`XX:XX:XX:XX:XX:XX`) |
> | `start_timestamp` | required | integer | Start of range (UNIX epoch) |
> | `end_timestamp` | required | integer | End of range (UNIX epoch) |

</details>

<details>
<summary><code>POST</code> <code><b>/mobile/export-vitals</b></code> <code>Export vital signs data (returns 501)</code></summary>
</details>

---

### Legacy Device API

These endpoints are from the original backend (`/http_api.py`). They use backend token authentication via the `Authorization` header.

#### Device Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/checkStatus2` | None | Get InfluxDB credentials for a device by MAC |
| POST | `/auth-device` | None | Authenticate device by MAC and token |
| POST | `/auth-user` | None | Authenticate user by name and token |
| POST | `/auth-forward` | None | Authenticate forward server |
| POST | `/auth-ai` | None | Authenticate AI server |
| POST | `/auth-admin` | None | Authenticate admin |

#### Vitals & Data

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/read-vitals-by-device-and-time` | User | Query vitals by device and time range |
| POST | `/read-device-list-by-user` | User | Get devices for a user |
| POST | `/read-device-list-by-ai` | AI | Get devices for AI server |
| POST | `/read-alert-list-by-ai` | AI | Get alerts for AI server |
| POST | `/read-device-list-by-forward` | Forward | Get devices for forward server |

#### Device Configuration

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/get-device-config-by-mac` | User | Get device config by MAC |
| POST | `/update-device-config-by-mac` | User | Update device config |
| POST | `/insert-device-config-by-mac` | User | Insert new device config |
| POST | `/read-influx-conf-by-forward` | Forward | Get InfluxDB config for forwarder |
| POST | `/read-mqtt-conf-by-forward` | Forward | Get MQTT config for forwarder |
| POST | `/read-mqtt-conf-by-ai` | AI | Get MQTT config for AI |
| POST | `/read-influx-conf-by-device` | Device | Get InfluxDB config for device |
| POST | `/read-tboard-conf-by-device` | Device | Get ThingsBoard config for device |

#### Device Management

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/register-license-by-device` | User | Register device licenses |
| POST | `/register-devices` | Admin | Bulk register devices |
| POST | `/activate-device-registration` | Admin | Launch background registration |
| POST | `/write-last-seen-by-device` | Forward | Record device last-seen timestamp |
| POST | `/set-beddot` | Admin | Configure BedDot device |
| POST | `/get-device-table` | User | Get device table data |
| POST | `/set-device-table` | User | Update device table |

#### Organization & Grafana

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/get-my-organization-name` | User | Get user's organization |
| POST | `/get-organization-table` | Admin | Get all organizations |
| POST | `/update-organization` | Admin | Update organization |
| POST | `/add-organization` | Admin | Create organization |
| POST | `/get-grafana-info-by-org` | Admin | Get Grafana info for org |
| POST | `/get-grafana_info-by-fullname` | Admin | Get Grafana user info |
| POST | `/refresh-grafana-dashboards` | User | Refresh Grafana dashboards |

---

## Integration Notes

Notes for mobile app developers integrating with this API.

**Password Reset is a 3-step flow:**
1. `POST /mobile/auth/reset-password` with `email` -- sends a recovery OTP
2. `POST /mobile/auth/verify-otp` with `email`, `otp_code`, `otp_type: "recovery"` -- verifies the OTP and unlocks the password update
3. `POST /mobile/auth/update-password` with `email`, `password` -- sets the new password (only works once after step 2)

**Two auth systems for mobile endpoints:**
Profile endpoints (`/mobile/user/*`) use JWT from login. All other mobile endpoints (`/mobile/get-*`, `/mobile/invite-*`, etc.) use backend auth, which requires the `user_name` field in the JSON body and a backend token in the `Authorization` header. The `user_name` corresponds to the user record in the legacy `user_conf` table -- this is separate from the `mobile_users` email-based identity. Both identities are linked via the `user_id` foreign key on the `mobile_users` table.

**Field naming is `snake_case` throughout.** For example: `device_mac`, `group_id`, `mobile_user_id`, `sleep_goals`, `wake_time`. The mobile app should convert from camelCase if needed.

**Refresh token rotation:** Each call to `/mobile/auth/refresh-token` invalidates the old refresh token and returns a new one. The app must store the new `refresh_token` from the response, otherwise the next refresh will fail.

**OTP delivery is not yet implemented.** The backend generates OTPs and stores them in the database, but does not send emails. During development, you can read the OTP directly from the `mobile_users.otp_code` column. An email service integration is planned.

---

## Legacy Install (Non-Docker)

1. Clone the repo
2. Update `.env` with MySQL credentials
3. Install `requirements.txt`
4. Install MySQL:
```bash
sudo apt install mysql-server
sudo mysql
CREATE USER 'dotbackend'@'localhost' IDENTIFIED BY 'mysecretpassword';
```
5. Start the server:
```bash
gunicorn -c gunicorn_config.py main:app
```

---

## Machine Code Conversion

The Dockerfile uses Cython to compile Python source to `.so` machine code during build. The entry point `http_api_server_main.py` is excluded from compilation since it is the gunicorn entry point.

To convert Python code manually:
1. Back up your source code
2. Copy `./packing_code/pack_setup.py` to `~/packing_code/`
3. From the target directory, run:
```bash
python3 ~/packing_code/pack_setup.py build_ext --inplace --include-dirs=/usr/include/python3.10
```
