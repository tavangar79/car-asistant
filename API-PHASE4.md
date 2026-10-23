# Phase 4 API contract — History, Dashboard, Profile

The rules in `DOMAIN.md` and the general rules of `API-PHASE2.md` / `API-PHASE3.md` still apply: Bearer auth on every route,
**404** for another user's vehicle or a malformed ObjectId, and **400** `{ "error": { "message", "details": { "<field>": ["…"] } } }`
for an invalid query. Money is integer toman. Dates are ISO-8601 UTC. An absent optional value is `null`.

Stitch screens (project `1400264005215370753`): history `7abfacc704a5434bb3b4985615f27ba7`, filter
`8afcf4e49c6f4ddea799f796513424b4`, dashboard `8acbd7d603f54b068264028fb2e0dd85`, profile `4b009228766f48859e449e996af538b9`.
Raw HTML: `C:\Users\Mohamad\AppData\Local\Temp\claude\E--Projects-car-assistant\e930cbf2-cd03-4f9f-b412-4f783927a33f\scratchpad\stitch\{history,filter,dashboard,profile}.html`.

**Jalali calendar on the backend.** The dashboard needs Jalali year and month boundaries. Add the `jalaali-js` dependency.
Boundaries are computed in **Asia/Tehran at a fixed UTC+03:30** (Iran has had no DST since 2022). Put this in one pure
helper, `src/lib/jalali.ts`, with unit tests: `jalaliMonthStart(date)`, `jalaliYearStart(date)` and `addJalaliMonths`.

**Cost of a record**, used everywhere below: service = `totalCost` (sum of its items), fuel = `totalCost`, expense = `amount`.
Odometer readings are **not** history entries, because the Stitch history screen has no odometer rows.

New modules: `src/modules/history/` and `src/modules/dashboard/`. They have the same layering as the others (types/zod,
service, routes, colocated tests, in-memory fakes) and read through the existing repositories. If a repository is
missing a query (a date range across several vehicles, for example), add one to it. Don't use Prisma directly in the
service. Mount both under `/api`. New deps go through `AppDependencies`.

---

## T4.1 `GET /api/history`

### Query (zod, all optional)

| param | type | meaning |
|---|---|---|
| `vehicleIds` | comma-separated ObjectIds | Absent means all of the caller's vehicles. If any id isn't the caller's → **404**. |
| `type` | `all \| service \| fuel \| expense` (default `all`) | The tab row: همه موارد / سرویس و تعمیر / سوخت‌گیری / هزینه‌های جاری |
| `categories` | comma-separated, from `fluids, brakes, suspension, electrical, drivetrain, periodic, legal` | See the matching rules below |
| `hasAttachment` | `true \| false` | `true` keeps only services that have an invoice and expenses that have an attachment. Fuel is excluded. |
| `from`, `to` | ISO datetime | Inclusive range on the record date (`serviceDate` / `fueledAt` / `paidAt`). The `from > to` case → 400 `details.from` |
| `cursor` | opaque string | From the previous page's `nextCursor` |
| `limit` | int 1..100, default 30 | |

**Category matching** (only when `categories` is present):
- A service matches when **any** of its line items has a catalog category in the set.
- An expense matches when `legal` is in the set and its category is not `other`. `legal` covers both insurances,
  `inspection`, `toll` and `fine`.
- Fuel logs never match. They are dropped when `categories` is present.

### Response 200

```json
{
  "entries": [
    {
      "type": "service",
      "id": "6650…",
      "vehicleId": "664f…",
      "occurredAt": "2024-11-15T08:00:00.000Z",
      "cost": 1130000,
      "hasAttachment": true,
      "service": { "...": "full Service JSON exactly as in API-PHASE2.md" },
      "fuelLog": null,
      "expense": null
    },
    {
      "type": "fuel", "id": "…", "vehicleId": "…", "occurredAt": "…", "cost": 145500, "hasAttachment": false,
      "service": null, "fuelLog": { "...": "full FuelLog JSON incl. computed economy (API-PHASE3.md)" }, "expense": null
    },
    {
      "type": "expense", "id": "…", "vehicleId": "…", "occurredAt": "…", "cost": 5850000, "hasAttachment": true,
      "service": null, "fuelLog": null, "expense": { "...": "full Expense JSON (API-PHASE3.md)" }
    }
  ],
  "nextCursor": "MjAyNC0xMS0xNVQwODowMDowMC4wMDBaXzY2NTA…",
  "summary": {
    "totalCost": 14280000,
    "serviceCount": 8,
    "fuelVolume": 240.5,
    "entryCount": 23
  }
}
```

- Order: `occurredAt` desc, then `id` desc. `nextCursor` is `null` on the last page. The cursor is base64url of
  `<occurredAt ISO>_<id>`. Each collection is queried with `(date < d) OR (date = d AND id < id)`, `limit + 1` rows.
  The three result sets are merged and sliced. A malformed cursor → 400 `details.cursor`.
- `summary` is computed over the **whole filtered set**, not just the page. It is sent only on the first page (no
  `cursor`) and is `null` on later pages. `fuelVolume` sums the litres of non-`cng` logs, rounded to 1 decimal.
- `economy` on a fuel log must be computed from **all** of that vehicle's logs (`computeEconomy`), not only the page.
- Tests: each filter, a combined filter, pagination across mixed types with equal timestamps, 404 for a foreign
  vehicle, 400 for a bad query, and summary totals.

---

## T4.4 `GET /api/dashboard`

No query. It covers all of the caller's vehicles.

```json
{
  "dashboard": {
    "jalaliYear": 1403,
    "yearTotalCost": 24850000,
    "invoiceCount": 14,
    "attentionCount": 3,
    "vehicles": [
      { "vehicleId": "664f…", "attentionCount": 2, "yearTotalCost": 12000000 }
    ],
    "months": [
      {
        "jalaliYear": 1403, "jalaliMonth": 3,
        "from": "2024-05-20T20:30:00.000Z", "to": "2024-06-20T20:30:00.000Z",
        "totalCost": 800000,
        "byVehicle": [ { "vehicleId": "664f…", "totalCost": 800000 } ]
      }
    ]
  }
}
```

- `jalaliYear` / `yearTotalCost`: the current Jalali year (Tehran time). The total covers the cost of every record
  from `jalaliYearStart(now)` to now. ("هزینه کل امسال")
- `invoiceCount`: services with an invoice plus expenses with an attachment, all time. ("فاکتورها … ثبت‌شده")
- `attentionCount`: the count of `computeDue` entries with status `overdue` or `due_soon`, summed over all vehicles.
  ("نیازمند رسیدگی … مورد فوری") Reuse the existing due computation and don't duplicate it.
- `vehicles`: one row per vehicle, in garage order.
- `months`: exactly **6** entries for the last 6 Jalali months including the current one, **oldest first**. `from` is
  inclusive and `to` is exclusive. `byVehicle` lists every vehicle, including those with 0, in the same order as `vehicles`.
- With no vehicles, the response is all zeros with `vehicles: []`. The 6 months still have `byVehicle: []`.
- Tests: the jalali helper (Nowruz boundary, Esfand of a leap year), month bucketing at boundaries, the attention count,
  the invoice count, and the empty garage.

---

## T4.6 Profile

`GET /api/profile/stats` → 200

```json
{ "stats": { "vehicleCount": 2, "serviceCount": 18 } }
```

- `serviceCount` is the number of service visits across all of the caller's vehicles.
- The documents tile ("مدرک در کیف") comes in Phase 5. Leave it out for now, and don't send a fake 0.
- `GET /api/me` must expose `createdAt` on the user (for "عضو رسمی از …"). Add it if it's missing, and add the field to the client DTO.

---

## Client notes (T4.2 / T4.3 / T4.5 / T4.6)

- Features: `client/lib/features/history/` (history + filter) and `client/lib/features/dashboard/` (home), plus the profile
  page in the existing profile/auth feature area. Same layering as `services/`. DTOs mirror the JSON above exactly, with
  `fromJson` tests using these sample payloads. History entries **reuse** the existing Service/FuelLog/Expense DTOs.
- **History** (`/history` tab, replaces the stub). Build it from Stitch: the vehicle selector in the app bar, the tune
  icon that opens the filter, the vehicle header, the summary cards (cost / service count / fuel litres), the type tabs,
  the "فاکتوردار" chip (= `hasAttachment=true`) and the year chip. Group entries by **Jalali month on the client**. A
  group header shows the count and the cost total of the loaded entries in that month. Use infinite scroll with `nextCursor`.
  Rows: a service title is built from its item titles (catalog), and a fuel row shows the economy when it isn't null. Tap
  opens the service details page (exists). Fuel and expense rows have no details page yet, so they don't navigate.
  Leave out the "خروجی اکسل/PDF" button until T6.1 (or show it disabled). The footer shows "شروع تاریخچه …" with the vehicle's `createdAt`.
- **Filter** (sheet/page per Stitch): vehicle multi-select plus "همه خودروهای گاراژ". Category chips are the 6 catalog
  categories (titles and icons from `GET /api/service-items`) plus `legal` "هزینه‌های قانونی" (`receipt_long`), and
  "انتخاب همه". It has the attachment-only toggle and a date range: سال جاری (the current Jalali year start → now), ۶ ماه اخیر,
  همه زمان‌ها, and انتخاب دستی (a Jalali date picker; reuse the one from the service wizard). The "تعداد نتایج" line is
  `summary.entryCount` from a `limit=1` query on the pending filter. "پاک کردن" resets, and "اعمال فیلتر" applies.
  One filter state provider feeds the history query.
- **Dashboard** (home tab, replaces the interim Home and wires the existing UI shell): the garage strip, the year cost
  card ("فیلتر در تاریخچه ›" opens history with the current-year filter), the invoices card ("مشاهده اسناد ›" opens
  history with `hasAttachment=true`), and the attention card ("بررسی موارد ›" opens the due-services page of the first
  vehicle whose `attentionCount > 0`). It also has the 6-month stacked bar chart from `months[].byVehicle`, with the
  latest month's total and its per-vehicle breakdown, and the "بروزرسانی کیلومتر" quick card.
- **Profile** (`/profile` tab, replaces the stub): the avatar (existing upload), the name, the phone, "عضو رسمی از
  <Jalali month year of createdAt>", and edit (the existing complete-profile/edit flow, if one exists). It has the
  vehicle and service stat tiles from `/api/profile/stats` and logout (existing). Leave out the subscription card
  (T6.3) and the documents tile (Phase 5). Settings and legal rows that have no page yet are shown disabled. The version
  line comes from the app version.
