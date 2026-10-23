# Phase 6 API contract — Report export, ownership transfer, plans

The rules in `DOMAIN.md` and the general rules of the earlier contracts still apply: Bearer auth on every route,
**404** for another user's vehicle/record or a malformed ObjectId, **400** `{ "error": { "message", "details": { "<field>": ["…"] } } }`
for invalid input. Dates are ISO-8601 UTC. Money is integer toman. An absent optional value is `null`.

Stitch screens (project `1400264005215370753`): export `6a6e4f3f474549c88b2f94b6a9c0722b`,
transfer `62ba7567cd1f4215a962c377ce8fa66e`, plans `f5d932ab55044100874d23a52a20ede1`.
Raw HTML + PNG: `C:\Users\Mohamad\AppData\Local\Temp\claude\E--Projects-car-assistant\ead4a88f-f9a8-4944-9286-7b6a8791e099\scratchpad\stitch\{export,transfer,plans}.{html,png}`.
Always fetch the screen through the `stitch` MCP as well (the PNGs above are thumbnails).

Product decisions (user, 2026-10-23):
- Transfer: the buyer must **accept in-app**. The vehicle moves only on accept.
- No read-only archive for the seller. Instead, records created before the transfer are **locked** for the new owner.
  The seller exports a report (T6.1) before sending if they want a copy.
- Plans: **display-only**, nothing is enforced, no payments. Client-only.

---

## T6.1 Report export

The **client** builds both files (CSV and PDF). Reason: the `pdf` Dart package shapes Persian/RTL text correctly,
while server-side PDF libraries don't do bidi. The backend only serves the report data in one call.

### `GET /api/reports`

Same filter semantics as `GET /api/history` (API-PHASE4.md), but **not paginated** and with a multi-type filter.

| param | type | meaning |
|---|---|---|
| `vehicleIds` | comma-separated ObjectIds | Absent = all of the caller's vehicles. Any id not the caller's → **404**. |
| `types` | comma-separated from `service, fuel, expense` | Absent = all three. Empty or unknown value → 400 `details.types`. |
| `from`, `to` | ISO datetime, optional | Inclusive range on the record date. `from > to` → 400 `details.from`. |

Response 200:

```json
{
  "entries": [ { "...": "history entry JSON exactly as in API-PHASE4.md (type, id, vehicleId, occurredAt, cost, hasAttachment, service|fuelLog|expense)" } ],
  "summary": { "totalCost": 18450000, "serviceCount": 8, "fuelVolume": 240.5, "entryCount": 24 }
}
```

- Order: `occurredAt` **asc**, then `id` asc (a report reads oldest → newest).
- `summary` is computed exactly as in history. The export screen's preview line ("شامل N رکورد ثبت‌شده • مجموع …")
  reads `summary.entryCount` and `summary.totalCost`.
- Hard cap: more than **5,000** matching entries → **422** `{ "error": { "message": "بازه گزارش بیش از حد بزرگ است" } }`.
- Reuse the history module's repository queries and `computeEconomy`. No duplicated filtering logic: extract a shared
  function if needed.
- Tests: type subset, date range, vehicle scope, 404 foreign vehicle, 400 bad `types`, ascending order, the 422 cap
  (with a small injected cap in the test).

### Client (T6.1)

- Page **خروجی گزارش** at `/reports/export` (optional `?vehicleId=`), built from the Stitch screen.
- Entry points: the history page's export button (enable it, pass the selected vehicle) and the vehicle details
  "دریافت کارنامه و گزارش فاکتورها (PDF)" button if present (pass that vehicle).
- Vehicle scope: segmented "this vehicle" (the default vehicle, or the one passed in) / "همه خودروها (N)".
- Date range pills: ۳ ماه اخیر, ۶ ماه گذشته, سال جاری (current Jalali year start → now), کل سوابق (`from` = null).
  Plus "از تاریخ / تا تاریخ" Jalali pickers (reuse the existing picker).
- Content checkboxes: سرویس‌ها و تعمیرات (`service`), سوابق سوخت‌گیری (`fuel`), هزینه‌های جانبی (`expense`),
  "انتخاب همه". At least one must be checked, otherwise both download buttons are disabled.
- "تصاویر پیوست فاکتورها": PDF only. Appends each **image** attachment (jpeg/png; service invoice and expense
  attachment, via the existing download endpoints) on pages at the end. Non-image attachments are listed by file name.
- Preview line refreshes (debounced) whenever the scope, range or types change.
- **CSV**: UTF-8 **with BOM** (Excel shows Persian correctly), comma-separated, RFC 4180 quoting. Columns:
  `تاریخ` (Jalali `1403/07/15`), `خودرو` (nickname), `پلاک`, `نوع` (سرویس / سوخت / هزینه), `شرح`, `کیلومتر`, `مبلغ (تومان)`.
  `شرح`: service = item titles joined with "، " + workshop; fuel = fuel type title + volume + station; expense =
  category title + title. Last row: `جمع کل` with the total. Digits in CSV are ASCII (spreadsheets must parse them).
- **PDF**: `pdf` package, RTL, the app's Vazirmatn font as an asset. Header (app name, report range in Jalali, vehicle(s),
  generated date), the summary block, one table per vehicle with the same columns as the CSV, Persian digits, total row.
- File names: `car-report-<jalali yyyy-mm-dd>.pdf|csv`. Save/share with the same mechanism the documents feature
  already uses for downloads (works on web). On success show the Stitch toast.
- The CSV and PDF builders are **pure functions** (`entries + vehicles + catalog → bytes`) with unit tests.

---

## T6.2 Ownership transfer

### Prisma

```prisma
model VehicleTransfer {
  id          String    @id @default(auto()) @map("_id") @db.ObjectId
  vehicleId   String    @db.ObjectId
  fromUserId  String    @db.ObjectId
  toPhone     String    // normalized with normalizeIranianPhone (same as login)
  status      String    // pending | accepted | rejected | cancelled
  createdAt   DateTime  @default(now())
  respondedAt DateTime?
  @@index([vehicleId])
  @@index([toPhone, status])
  @@index([fromUserId, status])
  @@map("vehicle_transfers")
}

model Vehicle {
  // … existing fields
  ownedSince DateTime? // set on accepted transfer; null = owned since createdAt
}
```

### Transfer JSON

```json
{
  "transfer": {
    "id": "6710…",
    "vehicleId": "664f…",
    "toPhone": "09121234567",
    "status": "pending",
    "createdAt": "2024-11-15T08:00:00.000Z",
    "respondedAt": null,
    "vehicle": {
      "nickname": "تارا اتوماتیک", "makeId": "…", "modelId": "…", "trimId": null, "yearJalali": 1402,
      "plate": { "twoDigits": "72", "letter": "ی", "threeDigits": "345", "provinceCode": "68" },
      "mileageKm": 42300
    },
    "fromName": "علی رضایی"
  }
}
```

`fromName` = seller's `firstName lastName` trimmed, or `null` when both are empty. `vehicle` is a snapshot read at
response time (the buyer can't call `/api/vehicles/:id` before accepting).

### Endpoints

| method + path | who | success | errors |
|---|---|---|---|
| `POST /api/vehicles/:id/transfers` body `{ "phone": "0912…" }` | owner | **201** `{ transfer }` | 400 `details.phone` invalid phone · 422 `details.phone` "نمی‌توانید خودرو را به شماره خودتان منتقل کنید" · 409 a pending transfer already exists for this vehicle |
| `GET /api/transfers` | anyone | **200** `{ "incoming": [Transfer], "outgoing": [Transfer] }` — **pending only**, newest first. Incoming = `toPhone` equals the caller's phone. Outgoing = `fromUserId` is the caller. | — |
| `DELETE /api/transfers/:id` | seller | **204**, status → `cancelled` | 404 not the caller's pending outgoing transfer |
| `POST /api/transfers/:id/reject` | buyer | **204**, status → `rejected` | 404 not the caller's pending incoming transfer |
| `POST /api/transfers/:id/accept` | buyer | **200** `{ "vehicle": Vehicle JSON }` | 404 as above · 409 the vehicle no longer belongs to `fromUserId` (then mark the transfer `cancelled`) |

The buyer may not have an account when the request is created. The request simply waits; once they sign up with that
phone, `GET /api/transfers` shows it. No SMS is sent (out of scope).

### Accept — effects (in this order, each step idempotent so a retry after a crash is safe)

1. `Vehicle.userId` → buyer, `ownedSince` → now, `isDefault` → `true` if the buyer has no other vehicle, else `false`.
2. If the vehicle was the seller's default and the seller still has vehicles, make the seller's most recently created one default.
3. Delete the vehicle's **identity documents** (`vehicle_card`, `green_sheet`, `power_of_attorney`) and their files.
   Insurance and inspection documents move with the car.
4. Delete all `Notification` rows for this vehicle (they belong to the seller).
5. Transfer status → `accepted`, `respondedAt` → now.

Everything else (odometer readings, services + invoices, intervals, fuel logs, expenses + attachments, photo) stays on
the vehicle and therefore moves to the buyer.

Accept also returns **409** `{ "error": { "message": "خودرویی با همین پلاک در گاراژ شما وجود دارد" } }` when the buyer
already has a vehicle with the same plate (the transfer stays `pending`).

Vehicle JSON gains **`ownedSince`** (ISO string or `null`) everywhere it is returned.

### Locked records

- A service visit, fuel log or expense is **locked** when `vehicle.ownedSince != null && record.createdAt < vehicle.ownedSince`.
- Service, FuelLog and Expense JSON get a new boolean field **`locked`** (also inside history/report entries).
- PATCH/DELETE on a locked record, and any write to its invoice/attachment → **403**
  `{ "error": { "message": "این رکورد مربوط به مالک قبلی است و قابل ویرایش نیست" } }`.
- Odometer readings and documents are not locked.

Tests: create (happy, self-phone 422, duplicate 409, foreign vehicle 404, bad phone 400), list incoming/outgoing,
cancel/reject/accept permissions (only the right side), accept effects 1–5, the 409 stale case, locked → 403 on each
record type, `locked` in JSON.

### Client (T6.2)

- Page **انتقال مالکیت خودرو** at `/vehicles/:id/transfer`, built from the Stitch screen. Entry: the vehicle details
  `more_vert` menu → "انتقال مالکیت".
- Phone input (+۹۸), consent checkbox; submit enabled only when the phone is valid and the box is checked. A final
  confirm dialog before sending (destructive action).
- Text changes vs. Stitch (to stay truthful, layout unchanged):
  - Under the phone field: "درخواست انتقال در حساب کاربری این شماره نمایش داده می‌شود و پس از تایید خریدار انجام می‌شود."
  - Rule ۲: "پیش از ارسال درخواست، در صورت نیاز از صفحه «خروجی گزارش» یک نسخه PDF یا اکسل از سوابق تهیه کنید." with a link to `/reports/export?vehicleId=…`.
- If the vehicle already has a pending outgoing transfer, the page shows its status (phone, date) and a
  "لغو درخواست" button instead of the form.
- Buyer side (no Stitch design, use the design system): a banner at the top of the garage page and a pinned card at the
  top of the notifications page for each incoming transfer. Tapping opens a bottom sheet with the vehicle snapshot,
  `fromName`, and "پذیرش" / "رد درخواست". Accept refreshes the garage and opens the vehicle.
- Locked records: hide/disable edit and delete for `locked == true` (service details page, and anywhere else edit exists),
  with a small "مالک قبلی" label.

---

## T6.3 Plans (client only)

- Page **پلن‌ها و تعرفه‌ها** at `/plans`, built from the Stitch screen; all copy and prices verbatim, as static data in
  one Dart file (monthly / yearly prices from the screen).
- Monthly / yearly toggle works (prices and subtitles switch as in the screen script).
- Current plan is always **پایه** ("پلن فعلی شما"). Upgrade buttons show a snackbar "به‌زودی" and do nothing else.
- Entry: the profile page's subscription card (left out in T4.6). Build it from the profile Stitch screen now, showing "پایه".
