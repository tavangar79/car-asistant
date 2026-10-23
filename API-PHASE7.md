# Phase 7 API contract — Dynamic service catalog

The rules in `DOMAIN.md` and the earlier contracts still apply (Bearer auth, 404 for another user's vehicle or a
malformed ObjectId, 400 `{ "error": { "message", "details": { "<field>": ["…"] } } }`).

Decisions (user, 2026-10-23):
- Scope levels: **global → make → model → trim**, using the vehicle's existing `makeId` / `modelId` / `trimId`.
  No fuel/gearbox fields (trims already encode AT/MT).
- Admin editing is **API only** (plus a CLI script to grant the role). No admin UI in this phase (T7.5 deferred).

Goal: the 26 service items and the 8 interval defaults move from static files into MongoDB, can differ per
make/model/trim, and admins can edit them. Visits keep storing item ids as strings, so no existing record changes.

---

## Concepts

- A **scope** is `{ makeId, modelId, trimId }`, each nullable. All null = **global**. `modelId` requires `makeId`,
  `trimId` requires `modelId`, and the ids must exist and belong together in the vehicle catalog (`VEHICLE_CATALOG`),
  otherwise 400 `details.scope`.
- A row **matches** a vehicle when each non-null scope field equals the vehicle's field.
  **Specificity:** trim 3 > model 2 > make 1 > global 0.
- **Resolution** for a vehicle: group rows by `itemId` (or `intervalId`), keep the matching rows, pick the most
  specific one. That row **fully replaces** the definition for that vehicle. If the picked row is `disabled`, the
  item/interval is disabled for that vehicle.
- Categories stay **static in code** (the 6 from `DOMAIN.md`). A row's `categoryId` must be one of them.

## Prisma

```prisma
model ServiceItemDef {
  id         String   @id @default(auto()) @map("_id") @db.ObjectId
  itemId     String   // stable slug stored on visits; never renamed
  makeId     String?
  modelId    String?
  trimId     String?
  categoryId String
  title      String
  subtitle   String
  icon       String   // Material Symbols name
  sortOrder  Int
  disabled   Boolean  @default(false)
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
  @@unique([itemId, makeId, modelId, trimId])
  @@map("service_item_defs")
}

model IntervalDef {
  id            String   @id @default(auto()) @map("_id") @db.ObjectId
  intervalId    String   // stable slug stored on ServiceInterval overrides
  makeId        String?
  modelId       String?
  trimId        String?
  title         String
  description   String
  icon          String
  coversItemIds String[]
  km            Int      // default km, 500..200000
  months        Int      // default months, 1..120
  sortOrder     Int
  disabled      Boolean  @default(false)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  @@unique([intervalId, makeId, modelId, trimId])
  @@map("interval_defs")
}

model User {
  // … existing fields
  role String @default("user") // user | admin
}
```

## T7.1 Storage, resolution, migration

- New repositories for both collections (+ in-memory fakes). One pure function `resolveForVehicle(rows, vehicle)`
  shared by items and intervals, with unit tests: global only, make beats global, model beats make, trim beats model,
  a non-matching scoped row is ignored, a disabled override hides the item, ordering by `sortOrder` then id.
- Reads load all rows per request (the collections are tiny). No cache.
- **Seed:** the current `service-items.data.ts` and `service-interval.data.ts` content moves to
  `src/scripts/catalog-seed.data.ts` as **global** rows (`sortOrder` = today's order). `npm run catalog:seed` upserts
  global rows **only when missing** (never overwrites admin edits). The server also runs the same seed at startup
  when a collection is empty, so a fresh environment works without a manual step.
- The runtime static data files and `ServiceItemCatalog`'s static constructor input are removed; everything reads
  through the repositories. Tests seed the in-memory repos from `catalog-seed.data.ts`.

## T7.2 Public endpoints (resolved per vehicle)

### `GET /api/service-items?vehicleId=`

Same shape as today, plus `disabled` on each item:

```json
{ "categories": [ { "id": "fluids", "title": "…", "items": [
  { "id": "engine_oil", "title": "…", "subtitle": "…", "icon": "oil_barrel", "disabled": false }
] } ] }
```

- With `vehicleId`: the vehicle's resolved catalog, **including** disabled items (`disabled: true`) so old visits can
  show their titles. Foreign/malformed vehicle → 404. Categories with no items are omitted.
- Without `vehicleId`: a **title lookup** of every known `itemId`: for each, the least specific row (global if it
  exists). `disabled` reflects that row. Used by history, reports and anywhere several vehicles are shown.
- Item order within a category: `sortOrder`, then `itemId`.

### Intervals and due

- `GET/PUT/DELETE /api/vehicles/:id/intervals` and `GET /api/vehicles/:id/due`: same shapes as API-PHASE2.md, but
  over the vehicle's **resolved, non-disabled** intervals (no longer "always 8"). `defaultKm` / `defaultMonths` come
  from the resolved row. `PUT` with an id that isn't resolved for the vehicle → 400 `details.intervals`.
- Stored overrides for intervals that are no longer resolved are ignored (not deleted).
- Notifications and the dashboard attention count use the same resolved intervals (they already go through due).

### Validation on service visits

- Creating a visit, or adding/changing a line item, requires each **new** `itemId` to be resolved and not disabled
  for that vehicle → else 400 `details.lineItems`. Existing line items whose item is now disabled may stay unchanged
  (a PATCH that doesn't touch them succeeds).
- History category filters use the title-lookup catalog (categoryId of each item).

## T7.3 Admin

- `requireAdmin` middleware after `authenticate`: role ≠ `admin` → **403** `{ "error": { "message": "دسترسی ندارید" } }`.
- Scripts: `npm run admin:grant -- 0912…` and `npm run admin:revoke -- 0912…` (phone normalized like login; unknown
  phone → exit code 1 with a message).

### Items — `/api/admin/service-items`

| method + path | body | success | errors |
|---|---|---|---|
| `GET /` | — | 200 `{ "items": [Row] }`, every row in every scope, incl. disabled, ordered by `itemId`, specificity, `sortOrder` | — |
| `POST /` | `{ itemId, scope: {makeId, modelId, trimId}, categoryId, title, subtitle, icon, sortOrder, disabled? }` | 201 `{ "item": Row }` | 400 invalid (itemId `^[a-z0-9_]{2,40}$`, scope, categoryId) · 409 same `itemId` + scope exists |
| `PATCH /:id` | any of `categoryId, title, subtitle, icon, sortOrder, disabled` (itemId and scope are immutable) | 200 `{ "item": Row }` | 400 · 404 |
| `DELETE /:id` | — | 204 | 404 · 422 for a **global** row (disable it instead; visits reference the id) |

Row JSON: `{ "id", "itemId", "scope": { "makeId", "modelId", "trimId" }, "categoryId", "title", "subtitle", "icon", "sortOrder", "disabled", "createdAt", "updatedAt" }`.

### Intervals — `/api/admin/intervals`

Same five operations and rules, with fields `intervalId, scope, title, description, icon, coversItemIds, km, months,
sortOrder, disabled`. `coversItemIds`: non-empty, each must exist as an `itemId` in some row → else 400.
`km` 500..200000, `months` 1..120.

Tests: 403 for a normal user on every admin route, CRUD happy paths, 409 duplicate, 422 global delete, scope
validation (unknown make, model not under make, trim without model), and an end-to-end case: a model-scoped
disabled override hides an item from `GET /api/service-items?vehicleId=` for that model only.

---

## T7.4 Client

- The wizard's catalog call passes the vehicle: `GET /api/service-items?vehicleId=<id>` (catalog provider becomes a
  family keyed by vehicle id). Step 1 hides `disabled` items.
- Everywhere else that only needs titles (history, reports, notifications, dashboard) keeps the call **without**
  `vehicleId`.
- Service details and the line-item editor: a line item whose item is `disabled` for that vehicle is shown
  **read-only** (title + cost, no edit button; delete still allowed) and can't be picked when adding an item.
- DTO: `disabled` bool on the item, defaulting to `false` when absent. Interval pages need no change (shape unchanged,
  list length may vary — make sure nothing assumes 8).
- No admin UI.
