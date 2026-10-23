# Domain rules (T0.1)

Sources: PRD screen `56be4091…` and the Stitch screens for service step 1, interval tuning, due services, fuel and expense.
These rules are fixed for all phases. Change them here first, then in code.

## General
- Money: integer **toman**. Distances: integer **km**. Dates: ISO-8601 UTC on the wire. The client shows Jalali dates and Persian digits.
- Each record belongs to one vehicle that the caller owns. Another user's vehicle returns **404**.
- No made-up metrics. No health %, scores or "official inquiry" claims (PRD §6). If a value can't be computed, it's `null` and the UI says "قابل محاسبه نیست".

## Odometer
- `POST /odometer` must be **≥ current `Vehicle.mileageKm`**. Otherwise **422**.
- A service or fuel record with a higher odometer also raises `mileageKm`. Back-dated records with a lower odometer are allowed and leave `mileageKm` unchanged.
- Quick-adjust chips: odometer modal −500 / +500 / +1000; fuel +50 / +100 / +500.

## Service items catalog (static, `GET /api/service-items`)
6 categories, 25 items (from step 1, titles verbatim from the screen), plus `fuel_filter` (it appears only on the interval screen):

| category | itemIds |
|---|---|
| `fluids` روغن و مایعات | `engine_oil`, `oil_filter`, `air_filter`, `cabin_filter`, `coolant`, `brake_fluid`, `fuel_filter` |
| `brakes` ترمز و ایمنی | `front_brake_pads`, `rear_brake_pads`, `brake_disc_skim`, `brake_booster_hoses` |
| `suspension` جلوبندی و تعلیق | `wheel_alignment`, `ball_joints_bushings`, `shock_absorbers`, `wheel_balancing` |
| `electrical` برق، باتری و شمع | `battery`, `spark_plugs`, `alternator_starter`, `lights_fuses` |
| `drivetrain` گیربکس و انتقال قدرت | `gearbox_oil`, `clutch_kit`, `cv_boots` |
| `periodic` سرویس‌های دوره‌ای اصلی | `timing_belt_kit`, `drive_belt`, `injector_cleaning`, `full_torque_check` |

Each item has a Persian title, subtitle and Material icon, copied from the screen.

## Service intervals (T2.5)
There are 8 tracked intervals. A service visit that contains **any** item in `coversItemIds` resets that interval.

| intervalId | coversItemIds | factory default |
|---|---|---|
| `engine_oil` | engine_oil, oil_filter | 5,000 km / 6 mo |
| `air_filter` | air_filter, cabin_filter | 10,000 km / 6 mo |
| `fuel_filter` | fuel_filter | 10,000 km / 12 mo |
| `front_brake_pads` | front_brake_pads | 25,000 km / 18 mo |
| `spark_plugs` | spark_plugs | 30,000 km / 24 mo |
| `gearbox_oil` | gearbox_oil | 50,000 km / 36 mo |
| `timing_belt_kit` | timing_belt_kit | 60,000 km / 48 mo |
| `coolant` | coolant | 40,000 km / 24 mo |

- Each vehicle can override km and months. "Reset all" restores the factory defaults.
- **Baseline:** the latest service visit that covers the interval (its date and odometer). If there is none, use the vehicle's `createdAt` and initial mileage.
- **Due:** `dueKm = baselineKm + intervalKm` and `dueDate = baselineDate + months`. **Whichever comes first** wins.
- **Status:** compute it from km and from days, then take the worse of the two:
  - `overdue`: remaining ≤ 0 (red, "رسیدگی فوری")
  - `due_soon`: ≤ 500 km or ≤ 14 days (amber, "نزدیک به موعد")
  - `upcoming`: ≤ 1,500 km or ≤ 30 days (blue, "اطلاعیه")
  - `ok`: anything else (green)
- The response has `remainingKm` and `remainingDays`. Both are negative when overdue.
- The due-services screen opens after an odometer update and lists every interval that isn't `ok`.

## Service visit
- date, odometer, workshop name, technician (optional), notes, invoice photo (optional), next reminder (optional).
- Line items: `{ itemId, brand?, cost, notes?, nextDueKm?, nextDueMonths? }`. Step 3 has one amount field per item ("مبلغ قطعه و اجرت"), so `cost` is part + labor together. The total is the sum of `cost` across items, computed on the server.
- A line item's `nextDueKm` (km after this visit) / `nextDueMonths` is the sticker reminder from step 3. When set, it replaces the vehicle's interval for that one cycle in the due computation (see `API-PHASE2.md`).
- "Has invoice" means an invoice photo is attached. The history "فقط فاکتوردار" filter uses it.

## Fuel
- `fuelType`: `regular | super | cng | lpg`. Also volume (litres; m³ for CNG), total amount (toman), odometer, date, station, payment method, notes, `isFullFill`.
- **Economy (L/100 km)** is set only when this log is a full fill, an earlier full fill exists, and every log between them has the same `fuelType`:
  `economy = sum(volume of logs after the previous full fill, up to and including this one) / (odometer − previous full-fill odometer) × 100`.
  In every other case it is `null`.

## Expenses
- `category`: `third_party_insurance | body_insurance | fine | toll | inspection | other`.
- Also amount (required), title, reference number, paid date, optional `expiresAt` (default suggestion is +1 year for insurance/inspection), attachment (PDF/JPG/PNG ≤ 10 MB), payment method `online | pos`, issuer.
- When `expiresAt` is set, a reminder notification fires **30, 15 and 7 days** before it.

## Documents (Phase 5)
- `type`: `third_party_insurance | body_insurance | inspection | vehicle_card | green_sheet | power_of_attorney` (from the Stitch screen). Also title, issuer, number, issue date, optional expiry (null = permanent) and up to 4 files. Files are stored encrypted at rest (AES-256-GCM, PRD §7). See `API-PHASE5.md`.
- Days-remaining badge uses the same colours: expired → red; ≤ 15 days → amber; ≤ 30 days → blue; otherwise green.

## Notifications (Phase 5)
- They come from interval status changes (`due_soon` / `overdue`) and from expense or document expiry (30/15/7 days).
- "Mark all read" is supported. Notifications older than **90 days** are deleted.
- Generated lazily (no cron) when the user reads them; one per interval cycle and status, one per expiry threshold. In-app only (no push yet).

## Open questions (defaults above are used until the user decides)
1. The due-soon / upcoming thresholds (500 km/14 d, 1,500 km/30 d) are inferred from the sample numbers on the due screen.
2. `fuel_filter` is on the interval screen but missing from the step-1 grid. It's added to `fluids`.
3. Offline-first (PRD §7) is out of scope for now. All writes go online.
