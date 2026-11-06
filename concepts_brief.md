# Car Assistant Admin Web Panel

* **Author:** Mohamad Tavangar
* **Date:** 2026-11-06

## Overview

This is a browser-based admin panel for the Car Assistant system. It is a React + TypeScript + Vite + shadcn/ui app in a new repository, added to the manager root as a third git submodule `admin/`, next to `backend/` (Express 5, Prisma 6.19, MongoDB) and `client/` (Flutter). One operator signs in with a username, password and a 6-digit authenticator (TOTP) code. That account is separate from app users, who sign in with phone + SMS OTP. The panel lets the operator:

1. see sign-up and activity numbers,
2. browse stored data read-only,
3. look at one user and everything they own,
4. block or unblock a user and fix their name or birth date,
5. manage the service catalog and interval defaults.

The value: Phase 7 left catalog management reachable only by API calls and an `admin:grant` script, and there is no way to inspect or moderate users today short of opening the database directly. The panel closes deferred task T7.5 (PLAN.md:89).

Six reviewer lenses examined the concept: design, codebase details, zoom-out/YAGNI, comprehension, implementation readiness and structure. All six completed. The concept as written has five problems that would make it unsafe or self-contradictory if built literally. They are listed first below. Each has a small fix that keeps the user's goal of simplicity.

## Analysis

### Blockers (must be decided before any code is written)

**1. Two admin systems would exist side by side.**
- An admin concept already exists:
  - `User.role` (`user` | `admin`, backend/prisma/schema.prisma:339-340),
  - `requireAdmin` (backend/src/modules/auth/require-admin.ts), which reads the role from the database on every request,
  - the `admin:grant` / `admin:revoke` scripts (backend/src/scripts/admin-role.ts),
  - and `/api/admin` mounted as `authenticate + requireAdmin + catalogAdminRoutes` (backend/src/app.ts:176-181).
- A new, separate admin account would get 403 on the existing catalog API. That API looks the caller up in `users`, and the new admin is not a user.
- If both systems are kept, anyone who controls the SIM of a granted phone can still edit the catalog using a 30-day user token, with no password or TOTP. That undoes the reason for adding TOTP.
- All six reviewers raised this.
- **Fix:** the new admin account replaces the role-based admin in one cutover:
  - move `catalogAdminRoutes` behind the new admin guard;
  - delete `requireAdmin`, `user-role.ts`, `admin-role.ts` and the two npm scripts;
  - leave the `role` field in the schema, unused, for one release so a rollback still finds it;
  - mark API-PHASE7.md T7.3 as superseded and update the backend/CLAUDE.md command table.
- No Flutter code calls `/api/admin`. Two reviewers grepped and found none.

**2. Admin tokens and user tokens would be interchangeable.**
- `TokenService` (backend/src/modules/auth/token.service.ts) signs HS256 with one `JWT_SECRET` and only a `sub` claim. `verify` checks no audience or token type.
- An admin token signed the same way would pass `requireAuth` on every user route. For example, `POST /api/vehicles` would create a vehicle owned by the admin's id.
- A user token would pass any admin guard that only checks the signature.
- **Fix:**
  - Add a separate `ADMIN_JWT_SECRET` to backend/src/config/env.ts: at least 32 characters, must differ from `JWT_SECRET`, placeholder rejected.
  - Build a second `TokenService` instance that also sets and requires `aud: "admin"`, and inject it through `AppDependencies`.
  - Do **not** start requiring `aud` on user tokens. Existing user tokens have none, so every app user would be logged out, and the client's dev-token recipe would break.
  - Add tests for both directions: a user token on an admin route returns 401, and an admin token on a user route returns 401.
- Note that `JWT_SECRET` also keys the OTP hasher (server.ts:69-71). This is another reason to leave it alone.

**3. "Re-run the CLI to recover" does not recover anything.**
- With no revocation, resetting the password and TOTP leaves every admin token already issued valid for up to 10 days.
- The only real kill switch would be rotating the secret by hand. If the secret were shared with users, that would log out every user.
- **Fix:** there is only one admin row, so the admin guard loads it on every request (one lookup by id, the same cost as today's `requireAdmin`). It rejects any token issued before a `credentialsChangedAt` timestamp stored on that row. The CLI reset updates the timestamp.
  - A simpler variant: the CLI deletes and recreates the row, so the old `sub` no longer exists.
- This is not a revocation feature. It is what makes the reset do what the brief already claims it does.

**4. Blocking is incomplete without changes to OTP and to the mobile app.**
- `requireAuth` is stateless today (require-auth.ts:11-20). "Rejected on every request, including existing tokens" means it must become async and read the user.
- Checking only there is not enough:
  - `verifyOtp` (auth.service.ts:85-110) issues a fresh token to any existing user, so a blocked user just logs in again;
  - `otp/request` keeps sending paid SMS messages to blocked phones.
- On the client:
  - The Flutter app signs out on 401 (client/lib/core/network/auth_interceptor.dart:27-37).
  - It shows "Invalid or expired code" for 400 or 401 on verify (auth_repository_impl.dart:44).
  - It has no special handling for 403 (api_call.dart:32-39).
- So a 401 for blocked users leads to a sign-out followed by a misleading "wrong code" loop. A 403 leaves the app signed in, failing on every screen.
- **Fix:**
  - Add `User.blockedAt DateTime?`, where null means active. This needs no data migration.
  - Check it in `otp/request` (before the SMS is sent), in `otp/verify`, and in `requireAuth`.
  - All three return the same response: 403 `{ "error": { "message": "<Persian text>", "details": { "code": "user_blocked" } } }`. The message is Persian because app versions already installed will show `message` as is.
  - A token whose user no longer exists gets 401 from `requireAuth`.
  - Add a client task (mohammad): on `user_blocked`, clear the token and show an "account blocked, contact support" screen. That task ships in the same release as the backend change.
  - Write down that rolling back the backend unblocks everyone.

**5. "Browse every collection" leaks secrets and sends huge payloads.**
- Five collections hold raw `Bytes`:
  - `vehicle_photos`
  - `service_invoices`
  - `expense_attachments`
  - `user_avatars`
  - `vehicle_document_files`, which is AES-256-GCM ciphertext and readable only through DocumentService.
- Prisma serialises bytes as JSON number arrays, about 4x the file size. One page of invoices could be tens of MB.
- `otp_codes.codeHash` and `otp_requests` are authentication material. The new `admins` collection would expose its own password hash and TOTP secret.
- **Fix:**
  - Keep an explicit registry in one backend file, `modules/admin/collection-browser.*`. Each entry names the Prisma model and the fields to leave out: bytes always, shown instead as `contentType` and `sizeBytes`.
  - Exclude `admins`, `otp_codes` and `otp_requests` completely.
  - Add one unit test that fails when a Prisma model is neither in the registry nor on the excluded list, checked against `Prisma.dmmf.datamodel`. Then every schema change forces a decision, and the list cannot quietly go stale.
  - The registry also feeds the panel's menu (`GET /api/admin/collections`). The UI draws columns from returned field names, so it has no per-model column code.
  - Do not use `$runCommandRaw` for browsing.
- The zoom-out lens argues for cutting the browser from v1 instead (see Alternatives).

### Major issues

**6. Token lifetime and storage.**
- A 10-day token kept in the browser's `localStorage` gives 10 days of full read access (every phone number, plate, VIN and document number) and block power to anyone who copies it, or to any XSS bug.
- `TokenService` already takes `expiresIn` as a constructor argument, so a shorter admin expiry costs nothing.
- Four lenses recommended 8 to 24 hours.
- This stays an open risk unless the user changes the 10-day choice. If 10 days stays, the brief should state the exposure in plain words, and Blocker 3's fix becomes more important.

**7. Login hardening is unspecified.**
- **Endpoint:** `POST /api/admin/auth/login` takes `{username, password, totp}` and has its own strict per-IP limit, reusing `ipRateLimit` (for example 5 attempts per 15 minutes).
- **Errors:** a single generic 401 "Invalid credentials" for any failure. It never says which factor was wrong.
- **TOTP:** RFC 6238 with 30-second steps, 6 digits and a ±1 step window. Store `lastTotpStep` on the admin row to reject a code that was already used.
- **Libraries:** the backend has no password-hashing or TOTP library today (package.json).
  - Password: node:crypto `scrypt` with `timingSafeEqual` needs no new dependency.
  - TOTP: about 30 lines over node:crypto HMAC, or `otplib`.
  - Inject `now` so tests are deterministic. The RFC 6238 test vectors give known codes.
- **Logging:** failed and successful logins are logged with the IP. Today the only server log is `console.error` for 500s.

**8. CLI handling of secrets.**
- The password must come from a hidden prompt, never from argv, which ends up in shell history.
- The CLI prints the base32 secret and the `otpauth://` URI once, ideally also as a terminal QR code.
- It deletes all admin rows and creates one, so a different username cannot leave two admins.
- Put the logic in a pure, unit-tested `resetAdmin(...)`, following the existing script pattern (admin-role.ts, catalog-seed.ts).
- The CLI needs the production `DATABASE_URL` on the operator's machine. That is an operational fact to note.

**9. "All of a user's data" has no clear meaning and no size limit.**
- Only `Vehicle`, `Notification`, `VehicleTransfer.fromUserId` and `UserAvatar` link to a user directly. Everything else hangs off `vehicleId`.
- Ownership transfers move vehicles. The records stay with the vehicle, and `ownedSince` (schema.prisma:29-31) hides older records from the new owner. The previous owner keeps no link to them.
- A heavy user can have thousands of fuel logs.
- **Fix:**
  - `GET /api/admin/users/:id` returns the profile, `blockedAt`, the vehicles currently owned and per-type counts.
  - Record lists come from the collection browser with a `userId` or `vehicleId` filter and are paged.
  - Records from before `ownedSince` are labelled in the UI.
- Most existing services take `userId` as their first argument (VehicleService, HistoryService, ProfileService and others) and can be reused. One exception: `NotificationService.sync` writes data, so admin reads must not call it.

**10. Profile editing scope.**
- Phone is the unique login identity, and pending transfers refer to it (`VehicleTransfer.toPhone`). Changing it is a separate, risky operation.
- **Fix:** v1 edits only `firstName`, `lastName` and `birthDate`, as a partial PATCH. Unknown keys, including `phone` and `role`, get 400.
- The existing user schema `createUpdateProfileSchema` requires all three fields. Reuse its field rules, not the schema as is.
- Optional: an "avatar delete" action as a moderation tool.

**11. The meaning and cost of the stats.**
- **What is counted:** each record has two dates, when it was entered (`createdAt`) and when it happened (`serviceDate`, `fueledAt`, `paidAt`). Backfilled history makes the two series very different.
  - Decide: count by `createdAt`.
  - Deleting a vehicle cascade-deletes its records, so past numbers shrink later. Document this.
- **Index cost:** no stats model has a `createdAt` index (User, Vehicle, ServiceVisit, FuelLog, Expense, OdometerReading, VehicleDocument). Every dashboard load scans about 7 collections.
- **Grouping:** Prisma `groupBy` cannot bucket dates by day.
- **Fix options:**
  - Add `@@index([createdAt])` on the counted models.
  - Fetch `createdAt` values within a bounded range and bucket them in a pure, tested function, in Asia/Tehran time, which the code already uses (auth.types.ts:46).
  - Or use `aggregateRaw` with `$dateTrunc`, which needs MongoDB 5 or later.
- **Smaller v1:** the zoom-out lens says a single card of plain counts (totals, new users in the last 7 and 30 days) is enough while the user base is small. Charts can come later.

**12. Persian server errors in an English UI.**
- Backend messages are Persian (`ADMIN_ONLY = 'دسترسی ندارید'`, require-admin.ts:6), with some in English.
- The error body has no machine-readable code.
- **Fix:**
  - The panel maps HTTP status plus an optional `details.code` to its own strings. It never matches on message text.
  - New admin endpoints use English messages.
  - Adding `details.code` should be written into the shared error contract.
- **Open question:** should the panel simply be Persian/RTL from day one? The operator is likely Persian-speaking, and user content (names, workshops, notes) is Persian anyway.

**13. Fitting a third submodule into the workspace.**
- The root CLAUDE.md names only two projects in several places: the project table, the "don't edit code" rule, the graphify lines and `git add backend client`.
- No agent can build React. Only `john` (backend) and `mohammad` (Flutter) exist.
- **Needed:**
  - an `admin/` row in the table;
  - a root agent twin plus an `admin/.claude/agents/` copy;
  - `admin/CLAUDE.md`, a skill and a graphify graph;
  - a definition of done: `npm run lint && npm run typecheck && npm test && npm run build`;
  - a decision on whether the Stitch-first UI rule applies to the panel;
  - `git add backend client admin` in the pin workflow.
- The proposed submodule URL is SSH, while the other two are HTTPS. Fresh `--recurse-submodules` clones without an SSH key will fail.

**14. Where the contract lives.**
- **Contract:** create one living `API-ADMIN.md` at the root (auth, block rules, users, stats, collection policy, and a link to the Phase 7 catalog section). Don't spread it across phase files.
- **Plan:** add "Phase 8: Admin panel" to PLAN.md, with tasks split into backend, admin and client.
- **Shared types:** the admin app keeps its own DTOs per feature, mirroring the contract, the same way client DTOs work today. The admin API client checks responses with zod at the boundary, so drift fails loudly. No shared types package and no OpenAPI generator.

**15. Gaps in the catalog screen.**
- The catalog's scope picker needs make, model and trim names, but `/api/vehicle-catalog` accepts only user tokens. Add `GET /api/admin/vehicle-catalog` with the same body.
- The UI must explain the existing rules inline:
  - global rows can't be deleted (422), only disabled;
  - `itemId` and scope can't be changed;
  - the km and months limits.
- This is the only screen whose backend already exists. Build it first to prove auth, the HTTP client and the table and form patterns.

**16. No audit trail.**
- Nobody will be able to answer "who blocked this user, and when, and why".
- Minimum for v1:
  - `blockedAt` plus an optional `blockReason`, both shown in user detail;
  - a console log line for each block, unblock or edit, with the target id.
- A small `admin_actions` collection can come later.

### Nits

- **Pagination:** cursor pages on `_id` descending (`take: limit+1`, limit capped at 100, `{ items, nextCursor }`). Avoid `skip` and exact filtered counts. Filter only on indexed fields (`userId`, `vehicleId`, `phone`).
- **CORS:** already in place (backend/src/shared/cors.ts). Add the panel origin to `CORS_ORIGINS` and `.env.example`. Bearer tokens without cookies mean no CSRF exposure. CORS is not access control, though. An IP allowlist or basic auth on `/api/admin` at the reverse proxy costs no code. Set `TRUST_PROXY_HOPS` so the login rate limit sees real IPs.
- **New env var:** making `ADMIN_JWT_SECRET` required stops every existing environment from booting until it is set. Treat setting it as a deploy step.
- **i18n:** no library for one language.
  - Use one typed `src/i18n/en.ts` object and a tiny `t(key)` helper, with no string literals in JSX.
  - Use Tailwind logical classes (`ms-`/`me-`/`ps-`/`pe-`/`start`/`end`) and set `<html dir>`.
  - Format dates and numbers with `Intl`, so Jalali (`fa-IR-u-ca-persian`) can be switched on later.
  - Use `dir="auto"` on cells that show user text.
  - Some shadcn components hard-code left and right and will need edits when RTL arrives.
- **UX basics:**
  - Every list, chart and detail page needs an empty, a loading and an error state.
  - A 404 page reads "not found, maybe deleted".
  - A 401 sends the operator to login with "session expired" and returns them to the same page afterwards.
  - Block and unblock use a confirm dialog that names the user and says the effect.
  - A blocked badge appears in the list and in detail.
  - One set of formatters handles ids (short, with copy), dates, toman, plates and enum labels.
- **Backend layout:**
  - A new `modules/admin/` folder (admin-auth, admin-users, admin-stats, collection-browser) behind one `adminRoutes(deps)` function mounted once, so app.ts grows only a few lines.
  - The catalog code stays in `service-catalog/`.
- **Test churn:** injecting `UserRepository` into `requireAuth` touches every Supertest suite that signs tokens for users missing from the in-memory repository. Plan time for it.

## Goals vs. Non-Goals

* **In-Scope Goals:**
  * A single admin account (username + scrypt password + TOTP), created or reset by `npm run admin:setup`. It replaces the role-based admin.
  * Admin tokens fully separate from user tokens: their own secret plus `aud: "admin"`, 10-day expiry.
  * Block and unblock, enforced at OTP request, OTP verify and on every authenticated request, with a defined 403 `user_blocked` response and a matching Flutter "account blocked" screen.
  * A user list with phone and name search, and user detail (profile, vehicles, counts, paged record lists), plus a partial profile edit (first name, last name, birth date).
  * A read-only collection browser over an explicit, tested registry that never returns bytes or auth secrets.
  * Stats: at minimum totals and recent counts by `createdAt` in Asia/Tehran time. Time series if the user keeps them.
  * Catalog management over the existing API, plus `GET /api/admin/vehicle-catalog`.
  * English UI with strings in one dictionary and direction-neutral CSS.
  * Workspace wiring: submodule, agent twin, CLAUDE.md updates, API-ADMIN.md and the PLAN.md Phase 8.
* **Out-of-Scope Non-Goals:**
  * Plans, packages, purchases and payments.
  * More than one admin, roles or permissions.
  * Token revocation of any kind (including `credentialsChangedAt`), refresh tokens and session lists. Deferred by the author.
  * Editing a user's phone or role, and deleting users.
  * Viewing decrypted document files or downloading invoice and attachment bytes. This is pending a privacy decision.
  * An i18n library, a shared types package and OpenAPI generation.
  * A full audit log collection (v1.1 at most).

## Alternatives Weighed

* **Keep role=admin alongside the new account.** Rejected by the reviewers: it leaves a weaker path that bypasses TOTP, and two guards with conflicting meaning under `/api/admin`.
* **Reuse phone OTP + role=admin and build no separate account** (zoom-out option a). New backend auth work would be zero. Not adopted: the user wants an identity separate from app users, and SMS OTP is weak (SIM swap) for an account that can read all user data. Still open if the user prefers it.
* **Add a TOTP step only for role=admin users on the existing phone login** (zoom-out option b). This keeps one identity and adds one field and a script. It is a valid middle option, not yet chosen by the user.
* **Sign admin tokens with the shared `JWT_SECRET` and only an `aud` claim.** Rejected: it needs every user-token check to reject `aud=admin`, and a mistake there gives admin tokens user access. A separate secret fails safe.
* **Require `aud` on user tokens too.** Rejected: it logs out every current app user and breaks the client dev-token recipe.
* **Cut the collection browser and use MongoDB Compass, mongosh, Atlas Data Explorer or Prisma Studio** (zoom-out). Cheaper and covers ad-hoc inspection. Not adopted yet, because the user asked for the browser explicitly. It remains the fallback if the registry is judged too costly.
* **A generic browser built on `$runCommandRaw` or `listCollections`.** Rejected: no field policy (leaks bytes and secrets), and it returns extended JSON.
* **A browser driven only by Prisma DMMF at runtime with dynamic delegate calls** (structural). Partly adopted: DMMF is used only in a coverage test. The browsing itself uses an explicit registry, so field policy is visible in code.
* **Serve the admin SPA from the backend with `express.static`.** Rejected: it couples the two builds and breaks the rule that app.ts has no I/O. It also goes against the workspace's one-repo-per-project structure.
* **AdminJS in place of a custom panel.** Rejected: it fits poorly with Prisma on MongoDB.
* **react-i18next now.** Rejected for v1 as a speculative abstraction. Add it when Persian arrives.
* **`credentialsChangedAt` check so a CLI reset kills old admin tokens.** Deferred by author: single admin, simplicity first; to be added in a later phase.
* **Shorter admin token (8–24 h).** Rejected by author: one admin, 10 days kept for convenience.
* **A per-request cache for the blocked check.** Rejected: it weakens "rejected immediately", and one lookup by id per request is about 1 ms at this scale.
* **Counting records by event date (`serviceDate`, `fueledAt`).** Rejected: backfilled history would make the "activity" charts misleading.

## Key Decisions & Staging for Next Steps

**Decisions confirmed by the author (2026-11-06):**
1. **One admin system.** The username + password + TOTP (Google Authenticator) account is the only admin. It replaces `User.role` admin, `requireAdmin` and `admin:grant` / `admin:revoke` in one cutover. `role` stays in the schema, unused, for one release.
2. **Separate admin secret.** `ADMIN_JWT_SECRET` plus `aud: "admin"`. User tokens are unchanged.
3. **No revocation in v1, 10-day token kept.** There is one admin (the author). `credentialsChangedAt` is deferred to a later phase. Accepted risk: a leaked admin token stays valid for up to 10 days. The only kill switch is rotating `ADMIN_JWT_SECRET`, which affects only admin tokens.
4. **Blocking as proposed.** 403 with `details.code = "user_blocked"` and a Persian message, enforced in `otp/request`, `otp/verify` and `requireAuth`. A deleted user gets 401. Ships together with the Flutter "account blocked" screen (mohammad).
5. **Collection browser shows only needed data.** An explicit registry lists the fields shown for each collection. Excluded entirely: password hashes, the TOTP secret, OTP codes and requests, and raw bytes (files are shown as `contentType` and `sizeBytes` only). A DMMF coverage test makes sure no model goes unclassified. Cursor paging on `_id`.
6. **Stats** count by `createdAt`, in Asia/Tehran time, over a bounded range, and show time-series charts for the full system picture (new users, vehicles, records by type).
7. **Profile edit** covers first name, last name and birth date only.
8. **Login:** password via node:crypto scrypt. TOTP per RFC 6238 (30 s, 6 digits, ±1 step) with replay protection. Login rate-limited, one generic 401. `npm run admin:setup` prompts for the password (hidden) and prints the secret, the `otpauth://` URI and a terminal QR code.
9. **UI:** English, all strings in one dictionary, direction-neutral CSS. shadcn/ui, no Stitch.
10. **New agent** for `admin/` (React + TS + shadcn), with a root twin and a repo-local copy, following the `john`/`mohammad` pattern.

**Still open:**
- The admin agent's name.
- Submodule URL: SSH as given, or HTTPS like the other two (fresh clones without an SSH key fail on SSH).
- Should the admin ever see decrypted documents or invoice and attachment files? Default: no in v1.
- Hosting: the panel's domain, and whether to add an IP allowlist on `/api/admin` at the proxy.

**Proposed contract skeleton (to expand in API-ADMIN.md).** Every route except login needs `Authorization: Bearer <adminToken>`. Errors use `{ "error": { "message", "details"? } }`, and a malformed id returns 404.
- `POST /api/admin/auth/login` with `{ username, password, totp }`: 200 `{ accessToken, expiresAt, admin: { username } }`, or 400, 401 or 429.
- `GET /api/admin/auth/me`: 200 `{ admin: { username } }`, or 401.
- `GET /api/admin/stats?from&to&bucket`: 200 `{ range, totals, series }`, with dense buckets in Asia/Tehran time.
- `GET /api/admin/collections`: 200 `{ collections: [{ name, count }] }`.
- `GET /api/admin/collections/:name?cursor&limit&userId|vehicleId`: 200 `{ name, omittedFields, documents, nextCursor }`. An unknown name gets 404.
- `GET /api/admin/users?search&cursor&limit`: 200 `{ users: [AdminUser], nextCursor }`.
- `GET /api/admin/users/:id`: 200 `{ user, vehicles (with counts), transfers, notificationCount }`, or 404.
- `PATCH /api/admin/users/:id` with any of `{ firstName?, lastName?, birthDate? }`: 200 `{ user }`, or 400 or 404.
- `POST /api/admin/users/:id/block` and `/unblock`: 200 `{ user }`. Both can be repeated safely. Repeating a block keeps the original `blockedAt`.
- `GET /api/admin/vehicle-catalog`: the same body as `/api/vehicle-catalog`.
- `/api/admin/service-items` and `/api/admin/intervals`: unchanged (API-PHASE7.md), with the new guard.
- User API change: a blocked user gets 403 `details.code = "user_blocked"` on every authenticated route and on OTP request and verify.

**Schema additions:** `User.blockedAt DateTime?` (and an optional `blockReason String?`). A new `Admin` model (`@@map("admins")`) with these fields:
- `username` (unique)
- `passwordHash`
- `totpSecret` (optionally encrypted with lib/file-crypto.ts)
- `lastTotpStep`
- `credentialsChangedAt`
- `createdAt` and `updatedAt`

**Proposed task order:**
1. **Root (CTO):** settle the open questions; write API-ADMIN.md and PLAN.md Phase 8; create the admin agent twin; update the root CLAUDE.md. The user creates the repo and adds the submodule, since that is a git write.
2. **Backend (john):**
   - B1: schema changes and in-memory fakes.
   - B2: pure TOTP and password modules.
   - B3: AdminRepository and `admin:setup`.
   - B4: env vars, admin token service, guard, login and me.
   - B5: catalog cutover plus `/api/admin/vehicle-catalog`.
   - B6: blocked enforcement plus the test churn.
   - B7: users.
   - B8: collections.
   - B9: stats.
3. **Admin (new agent):**
   - A0: scaffold, with the definition of done.
   - A1: API client, login and guard.
   - A7: catalog screen, first, after B5.
   - Then A5 and A6 (users), A4 (browser) and A3 (dashboard), each after its backend slice. MSW lets the UI start from the contract early.
4. **Client (mohammad):** C1, the `user_blocked` handling, shipped in the same release as B6.
