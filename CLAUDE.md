# FieldStock — Build Specification

Field service inventory management system for a utility meter services company.
Tracks inventory from a central warehouse to field trucks, through consumption on work orders.
Replaces a fully manual paper-based process.

---

## Tech Stack

| Layer | Decision |
|---|---|
| Frontend | Vue 3 (Composition API) — mobile-responsive web app |
| Backend | Node.js — REST API |
| Database | PostgreSQL on AWS RDS |
| Hosting | AWS — consistent with existing infrastructure |
| Auth | JWT with refresh tokens, HTTP-only cookies, role-based access control (RBAC) |
| Barcode Scanning | Device camera via browser — QuaggaJS or ZXing-js |
| Barcode Generation | Code 128, auto-generated from internal item ID |
| Label Printing | PDF label output from the app, formatted for standard label stock |
| File Export | CSV and PDF for reports |

---

## Core Inventory Flow

Everything in this system supports this single chain:

```
WAREHOUSE → TRUCK → JOB (Work Order)
```

1. **Warehouse** — Stock is received and stored. Manager maintains catalog and quantity on hand.
2. **Truck** — Items are transferred from warehouse to a specific truck. A tech checks out the truck and takes ownership of its inventory.
3. **Job** — Tech consumes items during or after a job. Consumption is posted against a WO number entered by the tech.

---

## What This System Does NOT Do

- Generate or manage purchase orders
- Integrate with the external work order management system (WO numbers are free-text entered by the tech)
- Track serialized items
- Track lot numbers, batch numbers, or expiration dates
- Operate offline (future phase)

---

## User Roles

All data is scoped to `org_id`. All API endpoints enforce role server-side — frontend role checks are UI convenience only.

| Role | Who | Access |
|---|---|---|
| `super_admin` | Platform owner | Cross-org access, system config, all client orgs. Platform-level only. |
| `admin` | Client account owner | Full access within their org: catalog, inventory, users, all reports, cost data. |
| `warehouse_manager` | Warehouse staff | Receive stock, manage catalog quantities, process transfers, run reports, print labels, conduct audits. |
| `technician` | Field tech | Check in/out a truck, log consumption against a WO, view their own truck stock (including item costs). Cannot edit catalog or view financial summaries. |
| `read_only` | Reporting viewers | Dashboard and reports only. No transactions. |

---

## Data Model

### Entities

**Organization**
```
id, name, created_at
```
Multi-tenant root. All data is scoped to org_id.

---

**User**
```
id, org_id, name, email, password_hash, role, active, created_at
```
Roles: super_admin | admin | warehouse_manager | technician | read_only

---

**Item**
```
id, org_id, internal_id (auto-assigned), description, vendor_part_number,
barcode, unit_of_measure, cost, supplier_id, reorder_threshold, active, created_at
```
- `internal_id` is system-assigned and used to generate the barcode if none exists
- `barcode` can be auto-generated (Code 128 from internal_id) or mapped to an existing barcode
- Designed for 1,000s of SKUs — index on barcode, internal_id, description
- Deactivated items remain in transaction history but cannot receive new transactions

---

**Supplier**
```
id, org_id, name, contact, notes, active
```

---

**Location**
```
id, org_id, type (warehouse | truck), name, active
```
Both the warehouse and each truck are Locations. Add trucks or warehouses via the UI — no migrations needed.

---

**Truck**
```
id, org_id, location_id, name, license_plate, active
```
Each truck maps 1:1 to a Location of type `truck`.

---

**TruckAssignment**
```
id, truck_id, tech_user_id, checked_out_at, checked_in_at, notes
```
- Active assignment = `checked_in_at` is NULL
- Only one active assignment per truck at a time (enforced by the API)
- A tech must have an active assignment to log consumption
- Admin can force-close an assignment

---

**Stock**
```
id, location_id, item_id, quantity_on_hand, updated_at
```
One row per item per location. Warehouse and each truck each have their own stock rows.

---

**ParList**
```
id, truck_id, item_id, par_quantity
```
Standard stocking level per item per truck. Used in low-stock reporting.

---

**Transaction**
```
id, org_id, type, item_id, from_location_id, to_location_id, quantity,
unit_cost, work_order_number, performed_by (user_id), notes, created_at
```
Single append-only table for all inventory movement. Never deleted.
Index on: created_at, item_id, from_location_id, to_location_id, work_order_number, org_id.

---

### Transaction Types

| Type | Description |
|---|---|
| `RECEIVE` | New stock added to the warehouse from a supplier shipment |
| `TRANSFER_OUT` | Items moved out of a location (warehouse or truck) |
| `TRANSFER_IN` | Items received at a destination location |
| `CONSUME` | Items used on a job — requires a valid `work_order_number` |
| `RETURN` | Items returned from a truck to the warehouse |
| `ADJUSTMENT` | Manual quantity correction — requires notes |
| `AUDIT` | Cycle count result — records counted quantity and variance |

All stock quantity changes go through a service-layer DB transaction — no direct quantity writes outside the transaction service.

---

## Work Order Numbers

WO numbers follow the format `WO-######` (e.g., `WO-001234`).
Validate this format on the API before posting any `CONSUME` transaction.
Return a clear validation error if the format doesn't match — don't silently post with a bad WO number.

---

## Barcode Scanning

- Uses device camera via QuaggaJS or ZXing-js
- Supports Code 128 and Code 39
- If a scanned barcode is unrecognized: prompt to assign to an existing item or create a new one
- Every scan field has a fallback: manual entry or type-ahead search
- Scanning is used in: receiving, transfers, consumption, cycle counts

---

## Feature Specifications

### Catalog Management (admin, warehouse_manager)
- Create, edit, deactivate items
- Auto-assign internal_id on create
- Auto-generate Code 128 barcode from internal_id if no barcode provided
- Print barcode labels: single item or bulk queue — PDF output for standard label stock
- Search/filter by description, internal_id, vendor part number, supplier, barcode
- Deactivated items stay in history, blocked from new transactions

### Receiving (warehouse_manager, admin)
- Multi-item session: scan or search items, enter quantity and unit cost, post all at once
- Posts `RECEIVE` transactions and increments warehouse stock
- Optional PO reference number (free text, not validated against any external system)
- Print labels for items received without barcodes

### Truck Check-In / Check-Out (technician, admin)
- Tech selects their name and a truck to check out → creates active TruckAssignment
- Only one active assignment per truck at a time — API enforces this
- Check-out records timestamp
- Check-in closes the assignment and records timestamp
- Tech can view current stock levels on the truck at check-out
- Admin can force-close an assignment
- Discrepancies in truck inventory are attributed to the tech who had it checked out

### Warehouse-to-Truck Transfers (warehouse_manager, admin)
- Select source (warehouse) and destination (truck)
- Scan or search items, enter quantity
- Validate quantity does not exceed source stock on hand
- Posts `TRANSFER_OUT` and `TRANSFER_IN` transactions, updates both location stock levels

### Truck-to-Truck Transfers (warehouse_manager, admin)
- Same flow as warehouse-to-truck with source = a truck location
- Formalizes informal borrowing between trucks

### Truck-to-Warehouse Returns (warehouse_manager, admin)
- Select truck as source, warehouse as destination
- Posts `TRANSFER_OUT` from truck and `TRANSFER_IN` to warehouse

### Consumption / Job Usage (technician) — PRIMARY MOBILE WORKFLOW
- Tech must have an active TruckAssignment to access this feature
- Tech enters a WO number (validated as `WO-######` format before session opens)
- Add items: scan barcode via camera, or search/select from truck's current stock
- Item cost is shown to the tech for each item as they log usage
- Enter quantity per item
- Session stays open until posted — items can be added throughout a job or all at end
- On post: `CONSUME` transactions created, truck stock decremented
- Warn if consumption would bring any item below zero on the truck
- Tech can add a note to the session (e.g., job address, meter ID)

### Par List Management (admin, warehouse_manager)
- Define par quantities per item per truck
- Clone a par list from one truck to another
- Par quantities used in low-stock reporting

### Inventory Adjustments (admin, warehouse_manager)
- Adjust quantity at any location
- Required fields: item, location, new quantity or delta, reason/notes
- Posts `ADJUSTMENT` transaction with user and timestamp

### Cycle Counts / Audits (admin, warehouse_manager)
- Create an audit session for a location
- System shows current on-hand as expected values
- User enters actual counted quantities
- System shows variance before posting
- Posting creates `AUDIT` transactions and updates stock to counted quantities
- Audit history retained for shrinkage reporting

---

## Reports

All reports available to `admin` and `warehouse_manager`.
Technicians can view their own truck stock only.
All tabular reports: CSV export. Key reports: PDF export.

| Report | Description |
|---|---|
| Current Stock by Location | All items and quantities at warehouse and each truck, with par levels |
| Low Stock / Reorder | Items at or below reorder threshold — manager's input for building a PO. CSV export. |
| Consumption by Work Order | All items consumed against a WO number — quantity, unit cost, extended cost, tech, truck, date |
| Consumption by Technician | Usage summary per tech over a date range |
| Usage Trends by Item | Consumption volume per SKU over time |
| Full Transaction History | All transactions — filterable by type, location, item, user, date range. CSV export. |
| Truck Stock Summary | All trucks: current assigned tech, total SKUs, items below par |
| Variance / Shrinkage | Expected vs. actual from audit sessions over time |
| Cost Summary | Total inventory value on hand by location |

---

## UI Notes

### Mobile (technician screens) — phone-first design
- Home: active truck status, quick actions (Log Usage, View My Truck Stock)
- Check Out Truck: select truck, confirm, view current stock
- Log Usage: WO number entry → scan or select items → quantities (with costs shown) → post
- My Truck Stock: item list with quantities for checked-out truck
- Check In: close active assignment
- Min touch target: 44px. No dense tables. Scan-first — minimize typing.

### Desktop (manager/admin screens)
- Dashboard: warehouse overview, truck status grid, low-stock alert count, recent transactions
- Catalog: item list with search/filter, inline or modal edit
- Receiving: session-based workflow
- Transfers: warehouse-to-truck and truck-to-truck
- Trucks: roster, current assignments, per-truck stock detail, par list editor
- Reports: date range + filter controls, export buttons
- Users: user management, role assignment, active/inactive toggle
- Audit: create and complete cycle count sessions

---

## Scalability

- All data scoped to `org_id` — row-level multi-tenant isolation
- Super admin queries across orgs; all other roles are org-scoped
- Location table is not hard-coded — add trucks or warehouses via UI
- Transaction table is append-only — never delete records
- Archive strategy for old records is a future concern

---

## Non-Functional Requirements

- JWT with refresh tokens, HTTP-only cookie storage, session expiry on inactivity
- All API endpoints enforce role server-side
- All stock changes go through a service-layer DB transaction
- Transactions are never deleted — full audit trail always intact
- Key tech screens (Log Usage, Truck Stock) target sub-2-second load on LTE
- Browser support: Safari iOS 15+, Chrome Android 90+, Chrome/Edge/Firefox current desktop

---

## Build Phases

### Phase 1 — Core (build this first)
Auth + users, catalog + barcode generation + label printing, warehouse stock, receiving, warehouse-to-truck transfers, truck check-in/check-out, consumption with WO posting and cost display, current stock report, low-stock report.

**Goal:** Replace the paper process. The core transaction chain is live.

### Phase 2 — Operations
Truck-to-truck transfers, truck-to-warehouse returns, par list management, inventory adjustments, transaction history report, consumption by WO report, consumption by tech report, CSV export.

**Goal:** Full operational visibility and accountability.

### Phase 3 — Audit & Insight
Cycle count / audit module, variance/shrinkage report, cost summary, usage trends by item, truck stock summary dashboard, PDF export.

**Goal:** Full feature parity with spec.

### Phase 4 — Future (do not build yet)
Offline sync, CSV catalog import, tech-initiated transfer requests, PO export, additional client onboarding.

---

## Open Items (resolved)

- **Odometer on check-in/check-out:** Not included in this version.
- **WO number format:** `WO-######` — validate on API before posting any CONSUME transaction.
- **Tech cost visibility:** Yes — technicians see item costs on the consumption screen.
- **Par lists:** Don't exist yet — manager builds them in the app post-launch.
- **Reorder thresholds:** Don't exist yet — manager sets them in the catalog post-launch.
- **Leftover item policy:** Managed by warehouse manager — returns are manager-initiated.
- **Barcode label stock:** Confirm label printer format with client before go-live (not a code decision).
