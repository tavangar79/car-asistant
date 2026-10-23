# Phase 2 API contract — Services

Rules from `DOMAIN.md` apply. Auth: `Authorization: Bearer <jwt>` on every route. Every route under
`/api/vehicles/:id` returns **404** when the vehicle is not the caller's (same as today). Validation
failures are **400** with `{ "error": { "message", "details": { "<field>": ["…"] } } }`. Business rule
failures are **422** with `{ "error": { "message", "details"? } }`. Money is integer toman, distance is
integer km, dates are ISO-8601 UTC strings. `null` is sent for absent optional fields (never omitted).

Stitch screens (project `1400264005215370753`): step 1 `60a05bbc6c0c47ae82f6ea393f737f69`, step 2
`8c226f9a9e26459086dc607051ce7b95`, step 3 `262e74911b5b45158bec636a3f484ef4`, service details
`ca8a1402532c42dcb345e55f5e855068`, intervals `dee2fcfb53034b6a96a868bb828e8d0f`, due
`5686e3f9f6cf403cb2aa717c13b9ccdc`. Downloaded HTML copies (text only, for titles/icons):
`C:\Users\Mohamad\AppData\Local\Temp\claude\E--Projects-car-assistant\3c039ec2-399b-4a6d-9ee8-cfb772c15c81\scratchpad\stitch\{step1,step2,step3,details,intervals,due}.html`

---

## T2.1 Service-items catalog (static, no DB)

`GET /api/service-items` → 200

```json
{
  "categories": [
    {
      "id": "fluids",
      "title": "روغن و مایعات",
      "items": [
        { "id": "engine_oil", "title": "تعویض روغن موتور", "subtitle": "<from step-1 screen>", "icon": "oil_barrel" }
      ]
    }
  ]
}
```

- Categories and item ids exactly as the table in `DOMAIN.md` (6 categories, 24 items + `fuel_filter` in `fluids`).
- `title`, `subtitle`, `icon` are copied from the step-1 HTML (`icon` = the Material Symbols name used there,
  e.g. `oil_barrel`, `filter_alt`). `fuel_filter` is not on step 1: title `تعویض فیلتر بنزین`, subtitle
  `صافی سوخت و خطوط سوخت‌رسانی`, icon `local_gas_station`.
- Module `src/modules/service-items/` mirroring `vehicle-catalog` (data file + class + routes). The class
  exposes `has(itemId)` / `get(itemId)` for the services module.

---

## T2.2 Service visits

### Model (Prisma, Mongo)

```prisma
model ServiceVisit {
  id          String            @id @default(auto()) @map("_id") @db.ObjectId
  vehicleId   String            @db.ObjectId
  serviceDate DateTime
  odometerKm  Int
  workshop    String
  technician  String?
  notes       String?
  lineItems   ServiceLineItem[]
  totalCost   Int               // sum of lineItems.cost, computed by the service
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  @@index([vehicleId, serviceDate])
  @@map("service_visits")
}
type ServiceLineItem {
  itemId        String
  brand         String?
  cost          Int      // part + labor together (one field on the screen)
  notes         String?
  nextDueKm     Int?     // reminder: km *after* odometerKm (delta), from the sticker
  nextDueMonths Int?     // reminder: months after serviceDate
}
model ServiceInvoice {           // one per visit, like VehiclePhoto
  id          String   @id @default(auto()) @map("_id") @db.ObjectId
  serviceId   String   @unique @db.ObjectId
  fileName    String
  contentType String
  data        Bytes
  updatedAt   DateTime @updatedAt
  @@map("service_invoices")
}
```

### Service JSON (used by every response below)

```json
{
  "id": "6650…",
  "vehicleId": "664f…",
  "serviceDate": "2025-11-15T10:30:00.000Z",
  "odometerKm": 34850,
  "workshop": "اتوسرویس البرز",
  "technician": "آقای محمدی",
  "notes": "تعویض کامل به همراه واشر کارتر",
  "lineItems": [
    { "itemId": "engine_oil", "brand": "بهران رانا 5W-40", "cost": 1850000, "notes": null, "nextDueKm": 7000, "nextDueMonths": 6 },
    { "itemId": "oil_filter", "brand": "سرکان", "cost": 280000, "notes": null, "nextDueKm": null, "nextDueMonths": null }
  ],
  "totalCost": 2130000,
  "invoice": { "fileName": "فاکتور.jpg", "contentType": "image/jpeg", "sizeBytes": 1400000, "updatedAt": "…" },
  "invoiceUrl": "/api/vehicles/664f…/services/6650…/invoice?v=1731670200000",
  "createdAt": "…",
  "updatedAt": "…"
}
```

`invoice` and `invoiceUrl` are `null` when no invoice is attached. `hasInvoice` (DOMAIN) = `invoice != null`.

### Routes (mount under `/api/vehicles`, behind `requireAuth`, like `odometerRoutes`)

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/:id/services` | `CreateService` | 201 `{ "service": Service }` |
| GET | `/:id/services` | — | 200 `{ "services": Service[] }` newest `serviceDate` first, max 50 |
| GET | `/:id/services/:serviceId` | — | 200 `{ "service": Service }` |
| PATCH | `/:id/services/:serviceId` | `UpdateService` (all fields optional; `lineItems` replaces the whole list) | 200 `{ "service": Service }` |
| DELETE | `/:id/services/:serviceId` | — | 204 |
| PUT | `/:id/services/:serviceId/invoice` | multipart, field `invoice`, jpeg/png/webp, ≤ 5 MB (reuse `imageUpload`) | 200 `{ "service": Service }` |
| GET | `/:id/services/:serviceId/invoice` | — | 200 image bytes, `Cache-Control: private, max-age=31536000` |
| DELETE | `/:id/services/:serviceId/invoice` | — | 200 `{ "service": Service }` |

`CreateService` (zod):

```json
{
  "serviceDate": "2025-11-15T10:30:00.000Z",   // required, ISO datetime, not in the future → 400 details.serviceDate (same as odometer recordedAt)
  "odometerKm": 34850,                           // required, int 0..MAX_MILEAGE_KM
  "workshop": "اتوسرویس البرز",                  // required, 1..100 chars, trimmed
  "technician": "آقای محمدی",                    // optional, ≤ 100
  "notes": "…",                                  // optional, ≤ 1000
  "lineItems": [                                 // required, 1..30 items, itemId unique within the list
    { "itemId": "engine_oil", "brand": "…", "cost": 1850000, "notes": "…", "nextDueKm": 7000, "nextDueMonths": 6 }
  ]
}
```

- `itemId` must exist in the catalog → otherwise 400 `details.lineItems`.
- `cost` int ≥ 0; `brand` ≤ 100; `notes` ≤ 500; `nextDueKm` int 100..200000; `nextDueMonths` int 1..120.
- A visit or service-id of another user's vehicle → 404. Malformed ObjectId → 404.
- **Mileage rule:** after create/update, if `odometerKm > vehicle.mileageKm`, record an odometer reading
  (`OdometerService.record` with `recordedAt = serviceDate`) so `Vehicle.mileageKm` rises. A lower value is
  allowed (back-dated visit) and leaves `mileageKm` unchanged. Delete never changes `mileageKm`.
- `VehicleService.remove` also deletes the vehicle's visits and invoices.

---

## T2.5 Service intervals and due items

### Interval catalog (static, backend code)

8 intervals from `DOMAIN.md`. Titles/icons from the intervals screen:

| id | title | icon | coversItemIds | default |
|---|---|---|---|---|
| `engine_oil` | روغن موتور و فیلتر روغن | `oil_barrel` | engine_oil, oil_filter | 5000 km / 6 mo |
| `air_filter` | فیلتر هوا و اتاق | `air` | air_filter, cabin_filter | 10000 / 6 |
| `fuel_filter` | فیلتر بنزین (صافی سوخت) | `local_gas_station` | fuel_filter | 10000 / 12 |
| `front_brake_pads` | لنت ترمز چرخ‌های جلو | `trip_origin` | front_brake_pads | 25000 / 18 |
| `spark_plugs` | شمع‌های جرقه‌زن موتور | `electric_bolt` | spark_plugs | 30000 / 24 |
| `gearbox_oil` | روغن گیربکس اتوماتیک (ATF) | `settings` | gearbox_oil | 50000 / 36 |
| `timing_belt_kit` | تسمه تایم و هرزگردها | `sync` | timing_belt_kit | 60000 / 48 |
| `coolant` | مایع خنک‌کننده (ضدیخ و آب رادیاتور) | `ac_unit` | coolant | 40000 / 24 |

Each also has a one-line Persian `description` (what the work is), written by John in the style of the due
screen's subtitles (e.g. engine_oil: `تعویض روغن موتور به همراه فیلتر روغن`).

### Model

```prisma
model ServiceInterval {        // only overrides are stored; a missing row means factory default
  id         String   @id @default(auto()) @map("_id") @db.ObjectId
  vehicleId  String   @db.ObjectId
  intervalId String
  km         Int
  months     Int
  updatedAt  DateTime @updatedAt
  @@unique([vehicleId, intervalId])
  @@map("service_intervals")
}
```

### Routes

`GET /api/vehicles/:id/intervals` → 200, always all 8, catalog order:

```json
{
  "intervals": [
    { "id": "engine_oil", "title": "…", "description": "…", "icon": "oil_barrel",
      "coversItemIds": ["engine_oil", "oil_filter"],
      "km": 7000, "months": 6, "defaultKm": 5000, "defaultMonths": 6, "isCustom": true }
  ]
}
```

`PUT /api/vehicles/:id/intervals` body `{ "intervals": [ { "id": "engine_oil", "km": 7000, "months": 6 } ] }`
→ 200 same shape as GET. Each listed interval is set; unlisted ones keep their value. `km` int 500..200000,
`months` int 1..120, unknown `id` → 400. Values equal to the defaults remove the override (`isCustom: false`).

`DELETE /api/vehicles/:id/intervals` → 200 same shape as GET, all overrides removed ("بازنشانی همه").

`GET /api/vehicles/:id/due` → 200, always all 8, sorted worst first (overdue, due_soon, upcoming, ok; within
a status by `remainingKm` ascending):

```json
{
  "asOf": "2025-11-15T10:30:00.000Z",
  "mileageKm": 34850,
  "items": [
    { "intervalId": "engine_oil", "title": "…", "description": "…", "icon": "oil_barrel",
      "coversItemIds": ["engine_oil", "oil_filter"],
      "status": "overdue",
      "baselineKm": 29500, "baselineDate": "2025-05-10T00:00:00.000Z", "baselineServiceId": "6650…",
      "dueKm": 34500, "dueDate": "2025-11-10T00:00:00.000Z",
      "remainingKm": -350, "remainingDays": -5 }
  ]
}
```

Computation (`now` injected, `asOf = now()`):

1. **Baseline** = the latest visit (by `serviceDate`) with a line item whose `itemId` ∈ `coversItemIds`
   → `baselineKm = visit.odometerKm`, `baselineDate = visit.serviceDate`, `baselineServiceId = visit.id`.
   None → the vehicle's earliest odometer reading (`mileageKm`, `recordedAt`) if one exists, else
   (`vehicle.mileageKm`, `vehicle.createdAt`); `baselineServiceId: null`.
2. **Interval** = the vehicle's override or the default. If the baseline visit's first covering line item has
   `nextDueKm` / `nextDueMonths`, those replace the km / months for this cycle (sticker reminder).
3. `dueKm = baselineKm + km`, `dueDate = baselineDate + months` (calendar months, UTC).
   `remainingKm = dueKm − vehicle.mileageKm`, `remainingDays = floor((dueDate − asOf) / 86400000)`.
4. **Status** = worse of km-status and day-status:
   `overdue` if remaining ≤ 0; `due_soon` if ≤ 500 km or ≤ 14 days; `upcoming` if ≤ 1500 km or ≤ 30 days; else `ok`.

Pure function `computeDue(...)` in its own file with unit tests (every status, both axes, override, no-baseline).

---

## T2.8 Step-3 polish (added 2026-10-21)

### Technician removed
`technician` is dropped everywhere: Prisma model, zod schemas, Service JSON, client DTOs/entities/UI. Existing
documents may still hold the field; Prisma ignores unknown fields, so no migration is needed.

### Suggestions from history

`GET /api/service-suggestions` → 200

```json
{
  "workshops": ["اتوسرویس البرز", "تعمیرگاه مرکزی"],
  "brands": { "engine_oil": ["بهران رانا 5W-40", "الف"], "oil_filter": ["سرکان"] }
}
```

- Scope: every visit of every vehicle of the caller. Distinct, trimmed, case-sensitive values, ordered by the most
  recent `serviceDate` they appeared in, max 10 per list. Items with no brands are omitted from `brands`.
- Optional `?q=` is NOT supported; the client filters locally.
- Mount at `/api/service-suggestions` behind `requireAuth` (not under a vehicle: suggestions span the garage).
