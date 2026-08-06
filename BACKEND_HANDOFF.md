# xStore Admin Dashboard — Backend Handoff

Front-end design prototype (no build step, plain HTML/CSS/JS). All data is currently
hard-coded in `src/app.js` and all actions are simulated (toasts/drawers). This doc lists
the endpoints and payload shapes the dashboard expects so the backend can wire it up.

Base path assumed: `/admin` (admin-authenticated). Currency is EGP, payment is Cash on Delivery.

## Files
- `src/index.html` — shell (sidebar, topbar, content container)
- `src/styles.css` — design tokens + all styles
- `src/app.js` — views, drawers, forms, and the in-memory demo data (`VENDORS`, `VLISTINGS`,
  `VCOMM`, `ORDERS`, `COURIERS`, `DISPUTES`, `CUSTOMERS`, `CATS`, `COUPONS`, `BANNERS`, `TEAM`)

Each in-memory constant in `app.js` maps 1:1 to a GET endpoint below. Replace the constant
with the response and the views render unchanged.

---

## Endpoints by view

### Overview  (`overview()`)
- `GET /admin/overview` → `{ gmv30d, orders30d, activeVendors, pendingApprovals, revenueTrend:[num], salesByCategory:[{name,count}] }`
- `GET /admin/orders?limit=5&sort=recent`
- `GET /admin/products?status=pending&limit=4`

### Product Moderation  (`moderation()`, `productDrawer()`)
- `GET /admin/products?status=pending`
- `POST /admin/products/{id}/approve`
- `POST /admin/products/{id}/reject`
- `POST /admin/products/{id}/request-changes`

### Vendors  (`vendors()`, `vendorDrawer()`, `vdecide()`)
- `GET /admin/vendors` → list of Vendor (see shape below)
- `GET /admin/vendors/{id}`
- `POST /admin/vendors/{id}/approve` · `.../reject` · `.../suspend` · `.../reinstate`

### Vendor detail page  (`vendorProducts()` — tabs: Listings / Orders / Commission)
- **Listings** — `GET /admin/vendors/{id}/products` → list of Listing
- **Orders** — `GET /admin/vendors/{id}/orders` → list of Order (buyer, items, total, status)
- **Commission** — see the Commission wallet section below

### Orders  (`orders()`, `orderDrawer()`)
- `GET /admin/orders?status=` · `GET /admin/orders/{id}`
- `POST /admin/orders/{id}/cancel`

### Delivery — couriers  (`couriers()`, `courierDrawer()`, `assignCourierDrawer()`)
- `GET  /admin/couriers` → list of Courier (see shape below)
- `POST /admin/couriers` `{ name, phone, zone }` — **owner-created account** with role
  `courier` (no self-registration path in the app; response should include the credentials
  flow, e.g. an invite SMS or temp password)
- `POST /admin/couriers/{id}/duty` `{ active: bool }` — on/off duty
- `POST /admin/couriers/{id}/cash-handover` `{ amountEgp }` — record COD cash deposited by
  the courier; subtract from `cashInHandEgp` (floor 0)
- `POST /admin/orders/{id}/assign-courier` `{ courierId }` — only for `confirmed|processing`
  orders without a courier; reject when the courier is off duty or at the cash cap
- App-side run list (already consumed by the Flutter app): `GET /orders/courier/{courierId}`

### Delivery Requests — package pilot  (`packages()`, `pkgDrawer()`, `assignPkgCourierDrawer()`)
Consumer point-to-point "send a package" flow. The customer submits a pickup + destination,
**admin sets the price**, the customer confirms (committing to pay the courier **in cash at
pickup** — the app never holds money), admin assigns a courier, the courier collects the cash
at pickup and delivers. Lifecycle: `submitted → priced → confirmed → pickedUp → delivered`
(`cancelled` allowed from `submitted`/`priced`).

- `POST /delivery-requests` `{ pickup, dropoff, packageNote }` — **called by the consumer app**;
  server sets `consumerId` from the token, status `submitted`, `price` null.
- `GET  /admin/delivery-requests?status=` · `GET /admin/delivery-requests/{id}` — admin queue.
- `GET  /delivery-requests/mine` — consumer's own requests (app "My Packages").
- `PUT  /admin/delivery-requests/{id}/price` `{ priceEgp }` — **admin only**; `submitted|priced
  → priced`. Re-pricing a `priced` request is allowed (customer hasn't confirmed yet).
- `POST /delivery-requests/{id}/confirm` — **consumer only**; `priced → confirmed`. This is the
  "customer pays" step (payment is cash-at-pickup, so no gateway call — just the commitment).
- `POST /admin/delivery-requests/{id}/assign-courier` `{ courierId }` — `confirmed` only; reject
  when the courier is off duty or at the cash cap (same rule as order assignment).
- `POST /delivery-requests/{id}/pickup` — **courier only**; `confirmed → pickedUp`. **On this
  transition the `priceEgp` must be added to the assigned courier's `cashInHandEgp`** (same
  ledger as COD; counts toward the handover cap). Cash changes hands here, not at drop-off.
- `POST /delivery-requests/{id}/deliver` — **courier only**; `pickedUp → delivered`.
- `POST /delivery-requests/{id}/cancel` `{ reason }` — consumer or admin; `submitted|priced` only.

> Mock pricing reference the prototype shows the admin (backend need not replicate): `60` base
> `+ 20` when pickup and dropoff cities differ. The real price is whatever the admin types.

> **Implemented (2026-08-06), real routes differ slightly from the spec above** — see
> `github.com/AhmedElemey/delivery-backend`'s own README for the authoritative contract
> (`/api/delivery-requests/*`, `/api/admin/*`, price body is `{price}` not `{priceEgp}`, cancel
> is requester-only not admin). Two additions this dashboard now uses:
> - `orderId` (nullable) on `DeliveryRequest` — set when a **vendor** raises the request against
>   an existing order whose pickup/drop-off doesn't fit the order's standard route. Shown as a
>   🔗 badge in the Delivery Requests table/drawer.
> - Vendor requests are **not** cash-at-pickup — their price is deducted from a vendor delivery
>   wallet instead, settled via `GET`/`POST /api/admin/vendors/{vendorId}/delivery-wallet[/settle]`
>   (derived from delivered, unsettled order-linked requests — same "sum of rows" pattern as the
>   courier cash wallet, no separate ledger table). Reachable from a vendor-raised request's
>   drawer → "Vendor delivery wallet".

### Disputes  (`disputes()`, `disputeDrawer()`)
- `GET /admin/disputes?status=`
- `POST /admin/disputes/{id}/resolve` body `{ resolution: "refund"|"partial"|"reject" }`

### Users / Customers  (`customers()`, `userDrawer()`)
- `GET /admin/users?role=consumer` · `GET /admin/users/{id}`
- `POST /admin/users/{id}/suspend`

### Categories  (`categories()`, `openCategoryForm()`)
- `GET /categories` (server-driven taxonomy) · `POST /admin/categories`
- `PATCH /admin/categories/{id}` `{ visible: bool }`

### Coupons  (`coupons()`)
- `GET /admin/coupons` · `POST /admin/coupons` · `PATCH /admin/coupons/{id}` · `DELETE /admin/coupons/{id}`

### Content & Banners  (`content()`)
- `GET /admin/banners` · `POST /admin/banners` · `PATCH /admin/banners/{id}` · `DELETE /admin/banners/{id}`
- `POST /admin/broadcasts` `{ title, audience, message }` (push)

### Settings  (`settings()`)
- `GET /admin/settings` · `PATCH /admin/settings` (marketplace policy toggles)
- `GET /admin/team` · `POST /admin/team/invite` `{ name, email, role }`

### Announcements
- `POST /admin/announcements` `{ title, audience, message }`

---

## Data shapes

### Vendor
```json
{
  "id": "v_123",
  "name": "Cairo Tech Hub",
  "owner": "Ahmed Salah",
  "city": "Giza",
  "category": "Electronics",
  "status": "active",          // active | pending | suspended
  "verified": true,
  "products": 214,
  "gmv": 486300,
  "rating": 4.7,
  "joined": "2026-01",
  "whatsapp": "+20 102 999 1122",
  "email": "shop@gizagadgets.eg"
}
```

### Listing  (vendor product)
```json
{
  "id": "p_456",
  "title": "iPhone 13 Pro 256GB",
  "subcategory": "Phones & tablets",
  "price": 48999,
  "compareAt": 52999,          // 0 / null if none
  "stock": 7,
  "status": "live",            // live | pending | out (out of stock)
  "sold30d": 42,
  "rating": 4.7
}
```

### Order
```json
{
  "id": "XS-2026-4471",
  "buyer": "Ahmed Hassan",
  "phone": "+20 100 111 2233",
  "address": "12 Tahrir St, Dokki, Giza",
  "vendorId": "v_123",
  "status": "delivered",       // pending|confirmed|processing|shipped|delivered|cancelled
  "payment": "cod",
  "courierId": "c_001",        // null = vendor self-delivery ("Delivered by xStore" pilot)
  "items": [ { "title": "iPhone 13 Pro 256GB", "qty": 1, "unitPrice": 48999 } ]
}
```

### Courier  ("Delivered by xStore" pilot)
Mirrors the Flutter entity `CourierCashWallet`
(`lib/features/delivery/domain/entities/courier_cash_wallet.dart`): COD the courier collected
and hasn't deposited yet. At/above `cashCapEgp` the courier must not receive new COD orders.
```json
{
  "id": "c_001",
  "name": "Mostafa El-Sayed",
  "phone": "+20 105 550 0003",
  "zone": "Cairo — Nasr City & Heliopolis",
  "status": "active",          // active | off (off duty)
  "cashInHandEgp": 3850.0,
  "cashCapEgp": 5000.0,        // maps to kCourierCashHandoverThresholdEgp (client const today)
  "tasksToday": 6,
  "delivered30d": 118,
  "failed30d": 7,
  "joined": "2026-06"
}
```

### DeliveryRequest  (package pilot)
Mirrors the Flutter entity `DeliveryRequestEntity`
(`lib/features/delivery/domain/entities/delivery_request.dart`). `price` is null until the admin
sets it; `courierId` is null until assignment (on `confirm` in the mock, but the real assign step
is admin-driven). Cash is collected from the **sender at pickup**, not the recipient at drop-off.
```json
{
  "id": "pkg_001",
  "consumerId": "u_456",
  "customer": "Sara Ahmed",      // sender; HIDDEN from the courier app until status >= confirmed
  "phone": "+20 106 000 0002",
  "pickup":  { "name": "Sara Ahmed", "phone": "…", "street": "15 Abbas El Akkad St", "city": "Nasr City" },
  "dropoff": { "name": "Laila Hassan", "phone": "…", "street": "8 El Merghany St", "city": "Heliopolis" },
  "packageNote": "Envelope with signed contract — handle with care",
  "priceEgp": 80.0,              // null until admin prices it
  "status": "confirmed",         // submitted|priced|confirmed|pickedUp|delivered|cancelled
  "courierId": "c_001",          // null until assigned
  "cancelReason": null,
  "createdAt": "2026-07-19T10:00:00Z"
}
```

---

## Commission wallet  (most important — newest feature)

Mirrors the Flutter entity `VendorCommissionWallet`
(`lib/features/commission/domain/entities/vendor_commission_wallet.dart`).
Platform takes a flat **2%** commission per COD order; unpaid commission accrues as the
vendor's **outstanding balance**, settled weekly. Thresholds are **per-vendor, admin-editable**.

### Shape
```json
{
  "outstandingEgp": 210.0,
  "warnThresholdEgp": 100.0,   // >= warn  -> notify vendor to pay      (alert: "warn")
  "pauseThresholdEgp": 200.0   // >= pause -> block new listing publishes (alert: "paused")
}
```
Alert level (server or client): `outstanding >= pause` → **paused**; else `>= warn` → **warn**; else **none**.

### Endpoints
- `GET  /admin/vendors/{id}/commission` → the wallet shape above
- `PATCH /admin/vendors/{id}/commission` `{ warnThresholdEgp, pauseThresholdEgp }`
  — save thresholds. Validate `pauseThresholdEgp >= warnThresholdEgp`.
- `POST /admin/vendors/{id}/commission/settle` `{ amountEgp }`
  — record a payment received from the vendor; subtract from `outstandingEgp` (floor 0).
  Omit `amountEgp` (or send the full balance) to **mark fully paid** (reset to 0).

> Note: the Flutter app currently hard-codes `kCommissionWarnThresholdEgp` /
> `kCommissionPauseThresholdEgp` as global consts. For per-vendor editing to work, the
> backend needs to store these per vendor and the app must read them from
> `GET /admin/vendors/{id}/commission` instead of the const.

---

## Wiring notes
- Every list view renders rows with `data-status="…"` — keep the status enum values above so
  the filter chips keep working.
- Money is rendered with `EGP(n)`; send raw numbers (EGP, no formatting).
- Actions currently call `toast(...)`; swap those for `fetch()` calls and re-render on success.
