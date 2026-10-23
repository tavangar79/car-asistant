# Remaining pages — plan & TODO

> On approval: copy this file to `E:\Projects\car-assistant\PLAN.md` (root, per user request). Not committed.

## Context
Built so far: Splash, Phone login, OTP, Complete profile, Add vehicle, Garage (list + empty).
Home = interim garage list; History / Profile = "Coming soon" stubs; notifications = TODO in `main_app_bar.dart:59`.
Backend has only auth, avatar, vehicle catalog, vehicles (+photo). No models for odometer, services, fuel, expenses, reminders, documents, notifications.
Stitch project `1400264005215370753` ("مدیریت خودرو Vehicle Ledger") has 18 undeveloped screens (below). PRD screen `56be409130514f699454992631358ad5` is the source for business rules — read it first.

## Working rules (each task)
- Fetch the Stitch screen via MCP; build exactly from it (no approximations).
- Cross-stack: CTO writes the API contract first → `john` builds backend → `mohammad` builds client DTOs to the real shape.
- Each task ships with tests; repo "definition of done" must pass. No commits.
- Error body: `{ "error": { "message", "details"? } }`. Auth: Bearer; all routes scoped to the owner's vehicle (404 for other users' vehicles).

## Phase 0 — Groundwork
- [x] T0.1 CTO: read PRD screen + DESIGN.md; settle domain rules → see `DOMAIN.md` (root).
- [x] T0.2 client: add `DESIGN.md` mirror to `client/` (required by client CLAUDE.md, missing).
- [x] T0.3 client: confirm Add-vehicle screen supports **edit mode** (PATCH `/api/vehicles/:id`); finish if missing.

## Phase 1 — Vehicle details & odometer (backbone for everything else)
- [x] T1.1 backend: `OdometerReading` model + `POST/GET /api/vehicles/:id/odometer` (monotonic check → 422; updates `Vehicle.mileageKm`).
- [x] T1.2 client: **بروزرسانی کیلومتر** (`b474a206…`) page + repository/provider.
- [x] T1.3 client: **جزئیات خودرو** (`50ada0c7…`) — vehicle header/plate, mileage, sections (sections with no data yet show empty states); route `/vehicles/:id`, tap from garage.
- [x] T1.4 client: **اقدامات سریع** bottom sheet (`8666dc1c…`) wired to center FAB; entries route to pages (unbuilt ones disabled until their phase).
- [x] T1.5 client: **گاراژ من** (`7e67f226…`) rebuilt per Stitch; set-default vehicle; dev user seeded with 3 vehicles + odometer history.

## Phase 2 — Services (core ledger)
Contract: `API-PHASE2.md` (root). Done 2026-10-21, uncommitted in both repos.
- [x] T2.1 backend: static service-items catalog `GET /api/service-items` (26 items, titles verbatim from step 1).
- [x] T2.2 backend: `ServiceVisit` + `ServiceInvoice` — CRUD + invoice under `/api/vehicles/:id/services`; higher odometer raises `mileageKm`.
- [x] T2.3 client: **ثبت سرویس** wizard — step 1 items (`60a05bbc…`), step 2 date+km (`8c226f9a…`), step 3 details/costs (`262e7491…`); one shared wizard state provider.
- [x] T2.4 client: **جزئیات سرویس** (`ca8a1402…`) — invoice card, edit/delete line items. Print/share/payment/PDF left out (not in contract).
- [x] T2.5 backend: `ServiceInterval` overrides + `GET/PUT/DELETE /api/vehicles/:id/intervals`; `GET /api/vehicles/:id/due` (pure `computeDue`, sticker override).
- [x] T2.6 client: **تنظیم دوره سرویس** (`dee2fcfb…`).
- [x] T2.7 client: **سرویس‌های موعد رسیده** (`5686e3f9…`) — opens after an odometer update; due section on vehicle details.
- Left open: no dev seed script exists (visits must be created through the app); garage card service banner still says "قابل محاسبه نیست"; "replace/remove invoice" not exposed in UI.
- [x] T2.8 step-3 polish (user feedback 2026-10-21; done same day, `GET /api/service-suggestions` added):
  - One workshop and one invoice image per visit, entered once in a shared card at the top of step 3. Remove the per-item copies and the "مشترک" tags.
  - Remove the technician field (**سرویس‌کار**) from the wizard, the details page and the API (`technician` dropped from `ServiceVisit`).
  - Workshop and per-item brand fields suggest previous values from the user's history (autocomplete, tap to fill).
    Backend: `GET /api/service-suggestions` → `{ "workshops": ["…"], "brands": { "<itemId>": ["…"] } }`, distinct values across the user's vehicles, most recent first, max 10 each.
  - Reuse the same shared card and suggestions in the details page's line-item editor.

## Phase 3 — Fuel & expenses
Contract: `API-PHASE3.md` (root). Done 2026-10-21, uncommitted in both repos.
- [x] T3.1 backend: `FuelLog` CRUD under `/api/vehicles/:id/fuel`; `economy` (L/100 km) computed on read by pure `computeEconomy` (full fill → previous full fill, same fuel type, else `null`); higher odometer raises `mileageKm`.
- [x] T3.2 client: **ثبت سوخت‌گیری** (`f9dfc5c4…`) at `/vehicles/:id/fuel/new`; local economy preview; fuel section on vehicle details; quick-actions row enabled.
- [x] T3.3 backend: `Expense` + `ExpenseAttachment` (pdf/jpeg/png ≤ 10 MB) under `/api/vehicles/:id/expenses`; categories `third_party_insurance | body_insurance | fine | toll | inspection | other`; `expiresAt > paidAt`; `expiryReminder` flag kept for Phase 5. Upload helper generalised (`file-upload.ts`), multer filenames now UTF-8.
- [x] T3.4 client: **ثبت هزینه** (`7ebb36d0…`) at `/vehicles/:id/expenses/new`; `file_picker` added for attachments; expense section on vehicle details.
- Left open: no edit/delete UI for fuel logs and expenses (API supports PATCH/DELETE); sample-only Stitch elements omitted (nozzle, tank capacity, standard rate); no "view" for a not-yet-uploaded attachment; graphify graphs in both repos are stale (`--update` needs an LLM key).

## Phase 4 — Home, History, Profile (replace stubs)
Contract: `API-PHASE4.md` (root). Done 2026-10-21, uncommitted in both repos.
- [x] T4.1 backend: unified timeline `GET /api/history?vehicleId&type&from&to&cursor` (services, fuel, expenses, odometer).
- [x] T4.2 client: **تاریخچه** (`7abfacc7…`) replaces stub.
- [x] T4.3 client: **فیلتر سوابق** (`8afcf4e4…`) sheet/page feeding T4.2 query.
- [x] T4.4 backend: `GET /api/dashboard` (per-vehicle due counts, recent records, monthly cost totals — counts only, no health scores).
- [x] T4.5 client: **داشبورد کلی گاراژ** (`8acbd7d6…`) replaces interim Home. UI shell done (Stitch layout, real vehicle/odometer data, empty states); wire data after T4.4.
- [x] T4.6 client: **پروفایل** (`4b009228…`) replaces stub (reuses `/me`, avatar, logout); `GET /api/profile/stats`.
- Left open: profile edit disabled (router only allows complete-profile while incomplete); history export button disabled until T6.1; fuel/expense rows have no details page; subscription card (T6.3) and documents tile (Phase 5) omitted; dev user has no records to show.

## Phase 5 — Notifications & documents
Contract: `API-PHASE5.md` (root). Done 2026-10-21, uncommitted in both repos.
- [x] T5.1 backend: `Notification` model, generated from due items/document expiry; `GET /api/notifications`, `PATCH /:id/read`.
- [x] T5.2 client: **اعلان‌ها** (`5b8dd2f6…`) + resolve TODO in `main_app_bar.dart:59`.
- [x] T5.3 backend: `VehicleDocument` (type, number, expiry, file bytes) CRUD.
- [x] T5.4 client: **کیف مدارک** (`ef4670d3…`).
- Left open: documents reachable only from garage tools row (default vehicle) + notifications — needs an entry on vehicle details/quick actions (no Stitch design); PDFs download instead of in-app view; no push notifications; `DOCUMENT_ENCRYPTION_KEY` must be set in every env.

## Phase 6 — Secondary
Contract: `API-PHASE6.md` (root). Decisions 2026-10-23: buyer accepts in-app; records before transfer are locked (no seller archive); plans display-only; PDF/CSV built on the client from `GET /api/reports`.
Done 2026-10-23, uncommitted in both repos.
- [x] T6.1 backend + client: **خروجی گزارش** (`6a6e4f3f…`) — PDF/CSV export of history for a date range.
- [x] T6.2 backend + client: **انتقال مالکیت خودرو** (`62ba7567…`) — transfer vehicle + records to another phone (destructive; confirm flow).
- [x] T6.3 client (+backend if plans are enforced): **پلن‌ها و تعرفه‌ها** (`f5d932ab…`) — display-only first; payments out of scope.
- Left open: not tested end-to-end (local Mongo not a replica set; `db:push` pending); pending transfers never expire; two concurrent transfer creates can both pass the 409 check; no SMS; no seller archive; PDF can wrap a word at a نیم‌فاصله at line end.

## Phase 7 — Dynamic service catalog (per car model / type)
Today the 26 service items and the 8 interval defaults are static backend data files. The client already renders
them from the API and visits store item ids as strings, so this phase changes no existing records or screens.
Contract: `API-PHASE7.md` (root). Decisions 2026-10-23: scopes global → make → model → trim (no fuel/gearbox); admin is API + CLI grant script only. Done 2026-10-23, uncommitted in both repos.
- [x] T7.1 backend: `ServiceItemDef` and `IntervalDef` collections with a scope (global → make → model → trim).
  Resolution picks the most specific match for a vehicle. Seed from the old static data (`npm run catalog:seed`, auto at startup when empty); static runtime files deleted.
- [x] T7.2 backend: `GET /api/service-items` and `GET …/intervals` resolve per vehicle (`?vehicleId=` for the catalog).
- [x] T7.3 backend: admin role on `User` + admin endpoints to add/edit/disable items and interval defaults per scope; `admin:grant` / `admin:revoke` scripts.
- [x] T7.4 client: pass the vehicle to the catalog call; show disabled items in old visits read-only.
- [ ] T7.5 (optional) admin UI: deferred by user decision (API only for now).
- Left open: `db:push` pending (new collections + unique indexes); not tested end-to-end; graphify graphs stale.

## Parallelism
Within a phase: backend task first when the client needs the new payload; client UI shells can start in parallel using the contract. Phases 3 and 5 are independent of each other after Phase 2.

## Verification (per task)
- backend: `npm test` + lint/typecheck in `backend/`; manual curl against the contract.
- client: `flutter analyze` + `flutter test` in `client/`; no integration tests.
- CTO review: `git -C backend diff`, `git -C client diff`, check contract match on both sides.
- End of each phase: start backend + client in Chrome (`NO_PROXY=localhost`, port 5555, dev auto-login) for the user to test.
