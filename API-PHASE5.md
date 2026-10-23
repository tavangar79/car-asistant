# Phase 5 API contract — Notifications & documents

The rules in `DOMAIN.md` and the general rules of the earlier contracts still apply: Bearer auth on every route,
**404** for another user's vehicle/record or a malformed ObjectId, **400** `{ "error": { "message", "details": { "<field>": ["…"] } } }`
for invalid input. Dates are ISO-8601 UTC. An absent optional value is `null`.

Stitch screens (project `1400264005215370753`): notifications `5b8dd2f6f8294f8e93ae15a912e4b372`,
documents `ef4670d3d43149de8878457fea948c23`.
Raw HTML + PNG: `C:\Users\Mohamad\AppData\Local\Temp\claude\E--Projects-car-assistant\976ce695-f583-4db9-8641-11cdde2a35a1\scratchpad\stitch\{notifications,documents}.{html,png}`.

New modules: `src/modules/documents/` and `src/modules/notifications/`, same layering as the others (types/zod,
service, routes, colocated tests, in-memory fakes). New deps go through `AppDependencies`. Order of work: T5.3 first
(documents), then T5.1 (notifications read documents and expenses).

---

## T5.3 Vehicle documents

### Types

| `type` | Persian title | Stitch filter group |
|---|---|---|
| `third_party_insurance` | بیمه شخص ثالث | `insurance` بیمه‌ها |
| `body_insurance` | بیمه بدنه | `insurance` |
| `inspection` | معاینه فنی | `inspection` معاینه فنی |
| `vehicle_card` | کارت خودرو | `identity` مدارک هویتی |
| `green_sheet` | برگ سبز (سند مالکیت) | `identity` |
| `power_of_attorney` | وکالت‌نامه | `identity` |

Grouping and titles live on the client. The server only validates the enum.

### Prisma

```prisma
model VehicleDocument {
  id             String    @id @default(auto()) @map("_id") @db.ObjectId
  vehicleId      String    @db.ObjectId
  type           String
  title          String?   // free text, e.g. "گواهی معاینه فنی برتر"; client falls back to the type title
  issuer         String?   // "بیمه ایران", "مرکز معاینه فنی شماره ۴ نیایش"
  number         String?   // policy / serial / VIN
  issuedAt       DateTime?
  expiresAt      DateTime? // null = permanent ("دائمی / بدون انقضا")
  expiryReminder Boolean   @default(true)
  notes          String?
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt
  @@index([vehicleId])
  @@map("vehicle_documents")
}

// Up to 4 files per document (front/back of a card, multi-page scans). Bytes are encrypted at rest.
model VehicleDocumentFile {
  id          String   @id @default(auto()) @map("_id") @db.ObjectId
  documentId  String   @db.ObjectId
  fileName    String
  contentType String   // application/pdf | image/jpeg | image/png
  sizeBytes   Int      // plaintext size
  data        Bytes    // AES-256-GCM: iv(12) | authTag(16) | ciphertext
  createdAt   DateTime @default(now())
  @@index([documentId])
  @@map("vehicle_document_files")
}
```

**Encryption (PRD §7).** One pure helper `src/lib/file-crypto.ts`: `encryptBytes(key, plain)` / `decryptBytes(key, blob)`,
AES-256-GCM with a random 12-byte IV, unit-tested (round trip, tampered blob throws, wrong key throws). The key is
`DOCUMENT_ENCRYPTION_KEY` (base64 of 32 bytes), read once in config and injected via `AppDependencies`. Startup fails
with a clear message if it's missing or not 32 bytes. Add it to `.env.example` and to the local `.env` (generate one).

### Document JSON

```json
{
  "id": "6651…",
  "vehicleId": "664f…",
  "type": "third_party_insurance",
  "title": null,
  "issuer": "بیمه ایران",
  "number": "124/987432",
  "issuedAt": "2023-12-01T00:00:00.000Z",
  "expiresAt": "2024-11-30T00:00:00.000Z",
  "expiryReminder": true,
  "notes": null,
  "files": [
    { "id": "6652…", "fileName": "policy.pdf", "contentType": "application/pdf", "sizeBytes": 1258291,
      "createdAt": "…", "url": "/api/vehicles/664f…/documents/6651…/files/6652…" }
  ],
  "createdAt": "…",
  "updatedAt": "…"
}
```

`files` is ordered by `createdAt` asc; `[]` when none.

### Routes (under `/api/vehicles/:id/documents`)

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/` | — | 200 `{ "documents": [Document] }` ordered: expiring soonest first (`expiresAt` asc, nulls last), then `createdAt` desc |
| POST | `/` | `CreateDocument` JSON | 201 `{ "document": Document }` |
| GET | `/:documentId` | — | 200 `{ "document": Document }` |
| PATCH | `/:documentId` | partial `CreateDocument` (nullable fields accept `null`) | 200 `{ "document": Document }` |
| DELETE | `/:documentId` | — | 204; also deletes its files and its notifications |
| POST | `/:documentId/files` | multipart, field `file`, pdf/jpeg/png, ≤ 10 MB | 201 `{ "document": Document }`; a 5th file → **422** `details.file` |
| GET | `/:documentId/files/:fileId` | — | 200 decrypted bytes, stored `Content-Type`, `Content-Disposition: inline; filename*=UTF-8''…`, `Cache-Control: private, max-age=31536000` |
| DELETE | `/:documentId/files/:fileId` | — | 200 `{ "document": Document }` |

`CreateDocument` (zod): `type` (enum, required), `title?` (≤ 100), `issuer?` (≤ 100), `number?` (≤ 50), `issuedAt?`,
`expiresAt?`, `expiryReminder?` (default true), `notes?` (≤ 500). Rule: when both dates are set, `expiresAt > issuedAt`
→ else 400 `details.expiresAt`. Upload reuses `file-upload.ts` (wrong type → 400 `details.file`, too large → 413).
New dep `documentFileMaxBytes` (default 10 MB). `VehicleService.remove` also deletes the vehicle's documents, files and notifications.

**Profile:** `GET /api/profile/stats` gains `documentCount` (documents across all the caller's vehicles):
`{ "stats": { "vehicleCount": 2, "serviceCount": 18, "documentCount": 5 } }`.

Tests: CRUD, the date rule, 404 for a foreign vehicle/document, file upload + download round trip (bytes in the DB are
not the plaintext), the 4-file limit, cascade on document and vehicle delete, `documentCount`.

---

## T5.1 Notifications

### Generation (no cron — lazy sync on read)

`NotificationService.sync(userId, now)` runs at the start of both GET routes below. It is idempotent:

1. Delete the user's notifications with `createdAt < now − 90 days`.
2. **Intervals:** for each vehicle, reuse the existing due computation (don't duplicate it). For each item with status
   `due_soon` or `overdue`, ensure one notification with key
   `interval:<vehicleId>:<intervalId>:<baselineDate ISO>:<status>`. A new covering service changes the baseline, so the
   next cycle notifies again; `due_soon` → `overdue` notifies again.
3. **Expiry** (expenses and documents with `expiryReminder = true` and `expiresAt` set):
   `daysLeft = ceil((expiresAt − now) / 1 day)`. If `0 < daysLeft ≤ 30`, `thresholdDays` = the smallest of `30, 15, 7`
   that is `≥ daysLeft`; ensure one notification with key `expense:<id>:<expiresAt ISO>:<thresholdDays>` (or `document:…`).
   Only the current threshold is created, never the skipped earlier ones. Expired items create nothing.
4. Put the pure part (which keys/payloads should exist given due items, expenses, documents and `now`) in one pure
   function `planNotifications(...)` with unit tests; the service only diffs against existing keys and inserts.

Uniqueness: `@@unique([userId, key])`. Deleting an expense or document deletes its notifications.

### Prisma

```prisma
model Notification {
  id            String    @id @default(auto()) @map("_id") @db.ObjectId
  userId        String    @db.ObjectId
  vehicleId     String    @db.ObjectId
  key           String
  kind          String    // interval_due_soon | interval_overdue | expense_expiry | document_expiry
  refId         String    // intervalId, expenseId or documentId
  subject       String    // snapshot title: interval title, expense title/category title, document title/type title (Persian)
  dueKm         Int?      // interval kinds
  remainingKm   Int?      // interval kinds, at creation
  mileageKm     Int?      // interval kinds, vehicle mileage at creation
  expiresAt     DateTime? // expiry kinds, and interval dueDate for interval kinds
  thresholdDays Int?      // expiry kinds: 30 | 15 | 7
  readAt        DateTime?
  createdAt     DateTime  @default(now())
  @@unique([userId, key])
  @@index([userId, createdAt])
  @@map("notifications")
}
```

For `subject` on expenses: the expense `title`, else the category's Persian title (`بیمه شخص ثالث`, `بیمه بدنه`,
`جریمه`, `عوارض`, `معاینه فنی`, `سایر`). For documents: `title`, else the type title from the table above.

### Notification JSON

```json
{
  "id": "6653…",
  "kind": "interval_due_soon",
  "category": "service",
  "vehicleId": "664f…",
  "refId": "engine_oil",
  "subject": "تعویض روغن موتور",
  "dueKm": 42000,
  "remainingKm": 150,
  "mileageKm": 41850,
  "expiresAt": "2024-12-01T00:00:00.000Z",
  "thresholdDays": null,
  "read": false,
  "createdAt": "…"
}
```

`category` is derived: interval kinds → `service`; expiry kinds → `documents`. (`system` exists only as a client tab and
is empty for now.)

### Routes (under `/api/notifications`)

| Method | Path | Response |
|---|---|---|
| GET | `/` | 200 `{ "notifications": [Notification], "unreadCount": 2 }` — newest first, max 100 |
| GET | `/unread-count` | 200 `{ "unreadCount": 2 }` (app-bar badge) |
| PATCH | `/:id/read` | 200 `{ "notification": Notification }` (idempotent) |
| POST | `/read-all` | 200 `{ "updatedCount": 2 }` |

Tests: `planNotifications` (each threshold, skipped thresholds, expired, reminder off, due_soon → overdue, new baseline),
sync idempotency (two syncs → no duplicates), 90-day purge, read / read-all scoped to the caller, 404 for another
user's notification, cascade on expense/document/vehicle delete.

---

## Client notes (T5.2 / T5.4)

- Features: `client/lib/features/notifications/` and `client/lib/features/documents/`, same layering as `expenses/`.
  DTOs mirror the JSON above exactly, with `fromJson` tests using these sample payloads.
- **Notifications** page (`/notifications`), built from Stitch: header "صندوق اعلانات", "N اعلان جدید", "خواندن همه"
  (`POST /read-all`), chips همه (count) / سرویس و نگهداری / بیمه و اسناد / سیستمی (client-side filter by `category`),
  unread rows tinted with a dot, the per-category empty state, and the footer "تمامی اعلانات به مدت ۹۰ روز…".
  Text is composed on the client from the fields, with Persian digits and the vehicle name from the garage:
  - interval: title "موعد <subject> <vehicle>", body "کارکرد خودرو به <mileageKm> کیلومتر رسیده است. تنها <remainingKm>
    کیلومتر تا سرویس باقی مانده است." (overdue: "<|remainingKm|> کیلومتر از موعد گذشته است."); action button "ثبت سرویس" → service wizard for that vehicle.
  - expiry: title "سررسید <subject>", body "<subject> خودرو <vehicle> <days> روز دیگر به اتمام می‌رسد."; chip "<days> روز باقیمانده"
    where `days` is computed from `expiresAt` now (not the snapshot).
  - Relative time ("۲ ساعت پیش", "دیروز، ۱۸:۳۰", "۳ روز پیش", "هفته گذشته") from `createdAt` — a pure, tested helper.
  - Tap: `PATCH /:id/read`, then navigate: interval → due-services page of the vehicle; document → documents page of
    the vehicle; expense → vehicle details.
  - Sample-only Stitch rows (fuel success, app update) are not generated — omit.
- **App bar bell** (all three: `main_app_bar.dart`, `home_app_bar.dart`, `profile_app_bar.dart`): opens `/notifications`
  and shows an unread badge from `GET /unread-count` (one shared provider, refreshed after read/read-all and on return).
- **Documents** page (`/vehicles/:id/documents`), built from Stitch: vehicle header + plate, the info line
  "N مدرک ثبت‌شده به‌صورت رمزگذاری‌شده نگهداری می‌شوند." (drop "در دستگاه شما" — files are on the server), the upload
  card (camera / gallery / PDF → opens the add form with the picked file), group chips with counts, and document cards:
  file-type badge (PDF/JPG/"N تصویر"), title, status line (expired → red "منقضی شده — N روز گذشته"; ≤ 15 d amber
  "موعد نزدیک — N روز مانده"; ≤ 30 d blue; else green "معتبر — N روز مانده"; null expiry "دائمی / بدون انقضا"),
  "issuer — شماره: number", "صدور: … • انقضا: …" in Jalali, a footer line "یادآور تمدید فعال است" when `expiryReminder`
  and an expiry is set, "مشاهده اصل فایل" (opens the first file; fetch bytes with the auth header), and "ثبت تمدید جدید"
  on expired cards (opens the add form prefilled with the same type/issuer).
  Omit the made-up footer claims (LEZ, VIN read, "آماده انتقال") and the "خروجی پرونده کامل" button (T6.1).
- **Add / edit document** form (bottom sheet or page; no Stitch screen — reuse the expense form's look): type, title,
  issuer, number, issue and expiry dates (the existing Jalali picker), reminder toggle, notes, files (reuse the
  `file_picker` flow from expenses; up to 4). Card tap opens edit, with delete.
- **Entry points:** enable the documents entry in the quick-actions sheet and on vehicle details if they exist; add the
  profile "مدرک در کیف" tile from `documentCount`. Replace the related `TODO(phase-5)` / `TODO(notifications)` comments.
