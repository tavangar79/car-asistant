# Admin API contract (Phase 8)

The source for the design decisions is `concepts_brief.md` (root), under "Decisions confirmed by the author".
This file is the living contract for the admin panel (`admin/`). The backend (`john`) builds to it, and the admin app (`mobin`) mirrors it exactly.

## General
- Base path: `/api/admin`. Every route except `POST /auth/login` needs `Authorization: Bearer <adminToken>`.
- Errors use `{ "error": { "message": string, "details"?: unknown } }`. Admin routes use English messages.
- Admin tokens: HS256 signed with `ADMIN_JWT_SECRET` (≥ 32 chars, must differ from `JWT_SECRET`, placeholder rejected), with `aud: "admin"` and `sub: <adminId>`. Expiry comes from `ADMIN_JWT_EXPIRES_IN` (default `10d`). There is no revocation (accepted risk).
- A user token on an admin route gets 401. An admin token on a user route gets 401. User tokens are unchanged: no `aud`, same secret.
- The admin guard verifies the token and loads the admin row by `sub`. A missing row gets 401.
- A malformed ObjectId in a path gets 404.
- Ids are 24-char hex strings. Timestamps are ISO-8601 UTC strings. Date-only values (`birthDate`) are `YYYY-MM-DD`.
- Cursor paging: `?cursor=<id>&limit=<1..100, default 25>`. Items are sorted by `_id` descending. The response has `nextCursor: string | null`.
- CORS: add the panel origin to `CORS_ORIGINS` (dev: `http://localhost:5173`).

## Cutover (removes the role-based admin)
- `/api/admin/service-items` and `/api/admin/intervals` (API-PHASE7.md) move behind the new admin guard. Their bodies are unchanged.
- Delete `requireAdmin`, `admin-role.ts`, the user-role helper, and the `admin:grant` / `admin:revoke` npm scripts.
- `User.role` stays in the schema, unused, for one release.

## CLI: `npm run admin:setup -- <username>`
- Prompts for the password twice, hidden. It is never read from argv. Minimum 12 characters.
- Deletes every admin row, then creates one with a new random TOTP secret (20 bytes, base32).
- Prints the base32 secret, the `otpauth://totp/CarAssistant:<username>?secret=…&issuer=CarAssistant&digits=6&period=30` URI, and a terminal QR code.
- The logic lives in a pure, tested `resetAdmin(...)`.

## Auth
### `POST /api/admin/auth/login`
Request body: `{ "username": string, "password": string, "totp": "123456" }`
- 200 `{ "accessToken": string, "expiresAt": ISO, "admin": { "username": string } }`
- 400 for a body that fails validation.
- 401 `"Invalid credentials"` for any failure: unknown user, wrong password, wrong or reused TOTP. It never says which factor failed.
- 429 from a per-IP limit of 5 attempts per 15 min (`ipRateLimit`).
- Password check: scrypt + `timingSafeEqual`.
- TOTP check: RFC 6238 SHA-1, 30 s steps, 6 digits, a ±1 step window. `lastTotpStep` is stored, so a step ≤ `lastTotpStep` is rejected.
- Every login attempt (success or failure) logs one line with the IP.

### `GET /api/admin/auth/me`
- 200 `{ "admin": { "username": string } }`
- 401

## Stats
### `GET /api/admin/stats?from=YYYY-MM-DD&to=YYYY-MM-DD&bucket=day|week|month`
- Defaults: the last 30 days, ending today, with `bucket=day`. The range is inclusive, at most 366 days. `from` must be ≤ `to`, otherwise 400.
- Dates are calendar days in Asia/Tehran. Weeks start on Saturday. A month bucket starts on the 1st (Gregorian).
- Records are counted by `createdAt`, not by event date. Deleted records are not counted, so past numbers can drop.
- 200 response:
```json
{
  "range": { "from": "2026-10-08", "to": "2026-11-06", "bucket": "day" },
  "totals": {
    "users": 0, "blockedUsers": 0, "vehicles": 0, "serviceVisits": 0,
    "fuelLogs": 0, "expenses": 0, "odometerReadings": 0, "documents": 0
  },
  "series": [
    { "bucketStart": "2026-10-08", "users": 0, "vehicles": 0, "serviceVisits": 0,
      "fuelLogs": 0, "expenses": 0, "odometerReadings": 0, "documents": 0 }
  ]
}
```
- `totals` are all-time counts. `series` is dense: every bucket in the range appears, including buckets with zero counts. Each bucket counts the rows created in it.
- Bucketing is a pure, tested function. Add `@@index([createdAt])` to every counted model.

## Users
`AdminUser`:
```json
{ "id": "…", "phone": "09120000000", "firstName": "…|null", "lastName": "…|null",
  "birthDate": "YYYY-MM-DD|null", "blockedAt": "ISO|null", "blockReason": "…|null",
  "createdAt": "ISO", "vehicleCount": 0 }
```
### `GET /api/admin/users?search=&cursor=&limit=`
- `search` matches part of the phone, or part of the first or last name, case-insensitive. It is optional.
- 200 `{ "users": [AdminUser], "nextCursor": string|null }`

### `GET /api/admin/users/:id`
- 200 response:
```json
{
  "user": AdminUser,
  "vehicles": [ { "...": "the vehicle as the `vehicles` collection shows it",
                  "counts": { "serviceVisits": 0, "fuelLogs": 0, "expenses": 0,
                              "odometerReadings": 0, "documents": 0 } } ],
  "transfers": [ "VehicleTransfer rows where fromUserId = id or toPhone = user.phone, as the `vehicle_transfers` collection shows them" ],
  "notificationCount": 0
}
```
- 404
- For record lists, the panel calls the collection browser with `vehicleId` or `userId`. It must not call `NotificationService.sync`, because that writes.

### `PATCH /api/admin/users/:id`
Request body: any of `{ "firstName"?: string, "lastName"?: string, "birthDate"?: "YYYY-MM-DD" }`. These use the same field rules as the profile.
- 200 `{ "user": AdminUser }`
- 400 for an empty body, an unknown key (`phone` and `role` included), or a value that breaks the field rules.
- 404

### `POST /api/admin/users/:id/block`
Request body: `{ "reason"?: string (≤ 500) }`
- 200 `{ "user": AdminUser }`
- 404
- Safe to repeat. Blocking a user who is already blocked keeps the original `blockedAt` and updates `blockReason`.

### `POST /api/admin/users/:id/unblock`
- 200 `{ "user": AdminUser }`, with `blockedAt` and `blockReason` set to null.
- 404
- Safe to repeat.

Every block, unblock and edit logs one line with the target id.

## User-side blocking (user API change)
- Schema: `User.blockedAt DateTime?` and `User.blockReason String?`. A null `blockedAt` means the user is active.
- `requireAuth` becomes async and loads the user by `sub` on every request:
  - The user is missing: 401.
  - The user is blocked: 403.
- `POST /api/auth/otp/request`: an existing phone that is blocked gets 403 before any SMS is sent.
- `POST /api/auth/otp/verify`: an existing phone that is blocked gets 403.
- The 403 body is always:
```json
{ "error": { "message": "حساب کاربری شما مسدود شده است. با پشتیبانی تماس بگیرید.",
             "details": { "code": "user_blocked" } } }
```
- Client (Flutter): on any 403 with `details.code == "user_blocked"`, clear the token and show an "account blocked" screen with the server message. That screen has one action, back to the phone login. Ship it in the same release as the backend change.

## Collection browser
### `GET /api/admin/collections`
- 200 `{ "collections": [ { "name": "users", "count": 0 } ] }`
- Names are the Prisma `@@map` names, in a fixed order.

### `GET /api/admin/collections/:name?cursor=&limit=&userId=&vehicleId=`
- 200 response:
```json
{ "name": "fuel_logs", "fields": ["id", "vehicleId", "..."],
  "documents": [ { "id": "…", "...": "…" } ], "nextCursor": null }
```
- 404 for an unknown or excluded name.
- 400 when `userId` or `vehicleId` is given for a collection that has no such field.
- `fields` lists the columns in display order. Every document is a flat object with only those keys. Nested objects or arrays pass through as JSON.
- An explicit registry in one backend file (`modules/admin/collection-browser.*`) lists the shown fields for each model.
- These are never returned:
  - the collections `admins`, `otp_codes` and `otp_requests`, which are excluded entirely;
  - any `Bytes` field. It is replaced by `sizeBytes: number`, and `contentType` is kept.
  - any hash, secret or ciphertext field.
- A unit test fails when a Prisma model (`Prisma.dmmf.datamodel`) is neither in the registry nor on the excluded list.

### `GET /api/admin/vehicle-catalog`
- The same body as the user route `GET /api/vehicle-catalog`, behind the admin guard. The catalog screen uses it for its scope picker.

## Env
- `ADMIN_JWT_SECRET`: required. Setting it is a deploy step.
- `ADMIN_JWT_EXPIRES_IN`: default `10d`.
- Both go in `.env.example`.
- The admin app reads `VITE_API_BASE_URL` (dev: `http://localhost:3000`).
