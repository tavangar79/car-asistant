# Phase 3 API contract — Fuel & expenses

Rules from `DOMAIN.md` and the general rules of `API-PHASE2.md` apply unchanged: Bearer auth on every route,
**404** for another user's vehicle or a malformed ObjectId, **400** `{ "error": { "message", "details": { "<field>": ["…"] } } }`
for validation, **422** `{ "error": { "message", "details"? } }` for business rules. Money is integer toman,
distance integer km, dates ISO-8601 UTC strings. Absent optional fields are sent as `null`, never omitted.
"Not in the future" uses the same `FUTURE_TOLERANCE_MS` as odometer/services.

Stitch screens (project `1400264005215370753`): fuel `f9dfc5c4c7644de1a68f8fef867f1be6`, expense
`7ebb36d09f6848dbb50a9a2f26264ab5`. Text-only HTML copies:
`C:\Users\Mohamad\AppData\Local\Temp\claude\E--Projects-car-assistant\b757766d-0a0f-4c29-a4a1-c9cc6530e794\scratchpad\stitch\{fuel,expense}.html`
(extracted labels/icons in `extract.txt` next to them).

Both modules mirror `service-visits`: `src/modules/fuel-logs/` and `src/modules/expenses/`, each with
`*.types.ts` (zod + TS), `*.repository.ts` (interface + Prisma), `*.service.ts`, `*.routes.ts`, colocated tests,
in-memory fakes under `src/test/`. Mounted under `/api/vehicles` next to `serviceVisitRoutes`. New deps go through
`AppDependencies`. `VehicleService.remove` also deletes the vehicle's fuel logs, expenses and attachments.

---

## T3.1 Fuel logs

### Model

```prisma
model FuelLog {
  id            String   @id @default(auto()) @map("_id") @db.ObjectId
  vehicleId     String   @db.ObjectId
  fueledAt      DateTime
  odometerKm    Int
  fuelType      String   // regular | super | cng | lpg
  volume        Float    // litres (m³ for cng), max 2 decimals
  totalCost     Int      // toman
  isFullFill    Boolean
  station       String?
  paymentMethod String?  // personal_fuel_card | station_fuel_card | bank_card | cash
  notes         String?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  @@index([vehicleId, fueledAt])
  @@map("fuel_logs")
}
```

### FuelLog JSON

```json
{
  "id": "6650…",
  "vehicleId": "664f…",
  "fueledAt": "2025-11-15T13:12:00.000Z",
  "odometerKm": 34850,
  "fuelType": "super",
  "volume": 48.5,
  "totalCost": 145500,
  "isFullFill": true,
  "station": "جایگاه اختصاصی ۲۴ یادگار امام (تهران)",
  "paymentMethod": "personal_fuel_card",
  "notes": "استفاده از اکتان بوستر کاسپین (۱ عدد)",
  "economy": 7.46,
  "createdAt": "…",
  "updatedAt": "…"
}
```

`economy` is litres per 100 km, rounded to 2 decimals, or `null` (see below). It is **computed on read**, never
stored, so edits and deletes of neighbouring logs stay consistent.

### Routes

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/:id/fuel` | `CreateFuelLog` | 201 `{ "fuelLog": FuelLog }` |
| GET | `/:id/fuel` | — | 200 `{ "fuelLogs": FuelLog[] }` newest `fueledAt` first, max 50 |
| GET | `/:id/fuel/:fuelLogId` | — | 200 `{ "fuelLog": FuelLog }` |
| PATCH | `/:id/fuel/:fuelLogId` | `UpdateFuelLog` (all fields optional) | 200 `{ "fuelLog": FuelLog }` |
| DELETE | `/:id/fuel/:fuelLogId` | — | 204 |

`CreateFuelLog` (zod):

```json
{
  "fueledAt": "2025-11-15T13:12:00.000Z",  // required, ISO datetime, not in the future → 400 details.fueledAt
  "odometerKm": 34850,                       // required, int 0..MAX_MILEAGE_KM
  "fuelType": "super",                       // required, enum regular | super | cng | lpg
  "volume": 48.5,                            // required, number > 0, ≤ 1000, at most 2 decimals
  "totalCost": 145500,                       // required, int ≥ 0
  "isFullFill": true,                        // required, boolean
  "station": "…",                            // optional, 1..100 trimmed
  "paymentMethod": "personal_fuel_card",     // optional, enum personal_fuel_card | station_fuel_card | bank_card | cash
  "notes": "…"                               // optional, ≤ 500
}
```

- **Mileage rule** (same as services): after create/update, if `odometerKm > vehicle.mileageKm`, call
  `OdometerService.record` with `recordedAt = fueledAt` so `Vehicle.mileageKm` rises. Lower values are allowed
  (back-dated log) and leave `mileageKm` unchanged. Delete never changes `mileageKm`.
- No uniqueness or monotonic checks between fuel logs.

### Economy — pure function `computeEconomy(logs)` in its own file with unit tests

Input: all logs of the vehicle. Output: `Map<id, number | null>`.

1. Sort ascending by `fueledAt`, then `odometerKm`, then `createdAt`.
2. For a log `L`, `economy` is set only when **all** hold:
   - `L.isFullFill` is true;
   - an earlier full fill `P` exists (the nearest one before `L` in the sorted order);
   - every log strictly between `P` and `L`, and `L` itself, has `fuelType === P.fuelType`;
   - `L.odometerKm − P.odometerKm > 0`.
3. Then `economy = round2( sum(volume of logs after P up to and including L) / (L.odometerKm − P.odometerKm) × 100 )`.
4. Otherwise `null`.

Tests: first log; partial fill; mixed fuel types in the window; zero/negative km delta; two consecutive full fills;
partials between two full fills (volumes summed); list order does not matter.

The client shows the same formula as a **preview** on the form ("میانگین مصرف این دوره") using the list it
already holds: most recent full fill before the entered odometer + the entered values. The saved value from the
server is authoritative.

---

## T3.3 Expenses

### Model

```prisma
model Expense {
  id              String   @id @default(auto()) @map("_id") @db.ObjectId
  vehicleId       String   @db.ObjectId
  category        String   // third_party_insurance | body_insurance | fine | toll | inspection | other
  amount          Int      // toman
  title           String?
  referenceNumber String?
  paidAt          DateTime
  expiresAt       DateTime?
  expiryReminder  Boolean  @default(true)  // the "ارسال اعلان یادآور" checkbox; used by Phase 5
  paymentMethod   String?  // online | pos
  issuer          String?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  @@index([vehicleId, paidAt])
  @@map("expenses")
}
model ExpenseAttachment {       // one per expense, like ServiceInvoice
  id          String   @id @default(auto()) @map("_id") @db.ObjectId
  expenseId   String   @unique @db.ObjectId
  fileName    String
  contentType String   // application/pdf | image/jpeg | image/png
  sizeBytes   Int
  data        Bytes
  updatedAt   DateTime @updatedAt
  @@map("expense_attachments")
}
```

Category catalog (static, shared by both stacks; the client hardcodes titles/icons, no endpoint):

| category | title | icon |
|---|---|---|
| `third_party_insurance` | بیمه شخص ثالث | `verified_user` |
| `body_insurance` | بیمه بدنه | `shield` |
| `fine` | جریمه رانندگی | `gavel` |
| `toll` | عوارض و طرح | `toll` |
| `inspection` | معاینه فنی | `fact_check` |
| `other` | سایر هزینه‌ها | `payments` |

### Expense JSON

```json
{
  "id": "6650…",
  "vehicleId": "664f…",
  "category": "third_party_insurance",
  "amount": 5850000,
  "title": "تمدید بیمه‌نامه شخص ثالث ۱۲ ماهه ایران",
  "referenceNumber": "1403/2480/991234",
  "paidAt": "2024-11-17T00:00:00.000Z",
  "expiresAt": "2025-11-17T00:00:00.000Z",
  "expiryReminder": true,
  "paymentMethod": "online",
  "issuer": "نمایندگی بیمه ایران (کد ۲۴۸۰)",
  "attachment": { "fileName": "بیمه‌نامه.pdf", "contentType": "application/pdf", "sizeBytes": 1258291, "updatedAt": "…" },
  "attachmentUrl": "/api/vehicles/664f…/expenses/6650…/attachment?v=1731844800000",
  "createdAt": "…",
  "updatedAt": "…"
}
```

`attachment` and `attachmentUrl` are `null` without a file.

### Routes

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/:id/expenses` | `CreateExpense` | 201 `{ "expense": Expense }` |
| GET | `/:id/expenses` | — | 200 `{ "expenses": Expense[] }` newest `paidAt` first, max 50 |
| GET | `/:id/expenses/:expenseId` | — | 200 `{ "expense": Expense }` |
| PATCH | `/:id/expenses/:expenseId` | `UpdateExpense` (all optional) | 200 `{ "expense": Expense }` |
| DELETE | `/:id/expenses/:expenseId` | — | 204 |
| PUT | `/:id/expenses/:expenseId/attachment` | multipart, field `attachment`, pdf/jpeg/png, ≤ 10 MB | 200 `{ "expense": Expense }` |
| GET | `/:id/expenses/:expenseId/attachment` | — | 200 file bytes, `Content-Type` as stored, `Cache-Control: private, max-age=31536000` |
| DELETE | `/:id/expenses/:expenseId/attachment` | — | 200 `{ "expense": Expense }` |

Upload: generalise the existing image upload helper to take an allow-list and a size limit (`imageUpload` keeps
its current behaviour for photos/invoices). New dep `expenseAttachmentMaxBytes` (default 10 MB) in
`AppDependencies`. Wrong type → 400 `details.attachment`; too large → 413 as today.

`CreateExpense` (zod):

```json
{
  "category": "third_party_insurance",   // required, enum above
  "amount": 5850000,                     // required, int ≥ 1
  "title": "…",                          // optional, 1..100 trimmed
  "referenceNumber": "…",                // optional, 1..60 trimmed
  "paidAt": "2024-11-17T00:00:00.000Z",  // required, ISO datetime, not in the future → 400 details.paidAt
  "expiresAt": "2025-11-17T00:00:00.000Z", // optional, ISO datetime, must be > paidAt → otherwise 400 details.expiresAt
  "expiryReminder": true,                // optional, boolean, default true
  "paymentMethod": "online",             // optional, enum online | pos
  "issuer": "…"                          // optional, 1..100 trimmed
}
```

- PATCH with `"expiresAt": null` clears the expiry. The `expiresAt > paidAt` check uses the merged record.
- Expenses never touch `Vehicle.mileageKm` (no odometer on the screen).

---

## Client notes (T3.2 / T3.4)

- Features `client/lib/features/fuel/` and `client/lib/features/expenses/`, same layering as `services/`
  (DTOs mirror the JSON above exactly; `fromJson` tests with the sample payloads).
- Routes `/vehicles/:id/fuel/new` and `/vehicles/:id/expenses/new` (top-level, like `services/new`).
- Quick-actions sheet: enable the fuel and expense rows (they need an active vehicle, like odometer/service).
- Vehicle details: the fuel and expense sections show the latest entries from the lists (empty state when none).
- After a successful save, refresh the vehicle so a raised `mileageKm` shows, and pop back.
- Fuel form labels for `paymentMethod`: `personal_fuel_card` → "کارت‌خوان جایگاه (کارت سوخت شخصی)",
  `station_fuel_card` → "کارت سوخت جایگاه (نرخ آزاد)", `bank_card` → "کارت بانکی", `cash` → "نقدی".
- Expense `expiresAt` default suggestion when the toggle is turned on: `paidAt + 1 year` for
  `third_party_insurance`, `body_insurance`, `inspection`; empty otherwise.
