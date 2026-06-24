# Configuration Guide

**Audience:** System administrators and Transport Finance. Covers the full setup sequence for a new installation.

---

## Setup Sequence

Follow this order — each step depends on the previous.

```
1. Company & CoA   →  2. Transport Setting  →  3. Locations  →  4. Routes
→  5. Trailer Types  →  6. Trailers & Tractors  →  7. Drivers
→  8. Rate Cards  →  9. Users & Roles  →  10. Test order
```

---

## 1. Company & Chart of Accounts

Enigma TMS uses ERPNext's standard company setup. Ensure:

- Company currency is **SAR**
- A complete Chart of Accounts exists (use the KSA CoA template)
- Fiscal Year 2026 (or current) is active
- **Required accounts:**
    - Freight Revenue (Income)
    - Fuel Expense (Expense)
    - Driver Pay Expense (Expense)
    - Toll Expense (Expense)
    - Trip Expense — Other (Expense)

---

## 2. Transport Setting

**Transport Setup → Transport Setting**

| Field | Recommended value |
|---|---|
| `apply_fuel_surcharge_on` | `Linehaul` |
| `default_vat_template` | KSA VAT 15% (create in ERPNext if not present) |
| `default_revenue_account` | Freight Revenue |
| `owned_fleet_cost_account` | Fuel Expense (or a clearing account if you split per type) |
| `auto_create_sales_invoice` | `Yes` for automated billing; `No` if Finance reviews first |
| `credit_check_on_order` | `Warn` initially; `Block` when credit process is mature |

---

## 3. Transport Locations

**Transport Setup → Transport Location → New**

Create one location per physical site: factories, warehouses, ports, customer sites.

| Field | Guidance |
|---|---|
| Location Name | Use a clear, unique name — it appears in route names |
| Location Type | Choose the most accurate type (Port, Factory, etc.) |
| City | Link to ERPNext City (creates city if not exists) |
| Country | Default: Saudi Arabia |
| Geolocation | Drop a map pin — required for future live-map feature |

**Minimum for go-live:** 2 locations (1 origin, 1 destination).

---

## 4. Transport Routes

**Transport Setup → Transport Route → New**

| Field | Guidance |
|---|---|
| From Location | Select origin |
| To Location | Select destination |
| Distance KM | Enter manually or leave blank if maps API is configured |
| Standard Transit Hours | Expected travel time; used for ETA calculations |

The `Route Name` field auto-fills as `{from} → {to}` — do not edit it.

!!! tip
    Routes are directional. If you operate in both directions (Riyadh → Jeddah **and** Jeddah → Riyadh), create two routes. Rate cards can have different prices per direction.

---

## 5. Trailer Types

**Transport Setup → Trailer Type → New**

Common types for a Saudi trailer fleet:

| Type Name | Capacity (tons) |
|---|---|
| Flatbed 40T | 40 |
| Lowbed Heavy | 60 |
| Reefer 30T | 30 |
| Curtain-Side 25T | 25 |
| Container Chassis 20ft | 25 |
| Box Van 20T | 20 |

---

## 6. Trailers & Tractors

### Trailers — **Transport Setup → Trailer → New**

| Field | Guidance |
|---|---|
| Plate Number | Official KSA plate; must be unique |
| Trailer Type | Link to Trailer Type created in step 5 |
| Ownership | `Owned` for your fleet; `Outsourced` for vendor trailers |
| Subcontractor | Required if Outsourced; select from ERPNext Supplier list |
| Capacity Tons | Physical capacity |
| Status | Default `Available`; the system updates this automatically |

### Tractors — **Transport Setup → Tractor → New**

Same structure as Trailer but for prime movers (truck heads). Link a Tractor to a Trip if you want to track prime mover utilization separately from the trailer.

---

## 7. Drivers

Drivers use the ERPNext core `Driver` doctype with custom fields added by this app.

**HR → Driver → New**

| Field | Notes |
|---|---|
| Full Name | As on license |
| `driver_type` | `Owned` or `Subcontractor` |
| `national_id_iqama` | Saudi national ID or Iqama number |
| `license_expiry` | Date — system warns when approaching expiry |
| `transporter` | Link → Supplier (if subcontractor driver) |

---

## 8. Rate Cards

See [Rate Cards](rate-cards.md) for full configuration details.

**Quick checklist:**

- [ ] Create a **default Customer Rate Card** (Customer = blank) covering all standard routes
- [ ] Create **customer-specific cards** where pricing differs from default
- [ ] Create a **default Subcontractor Rate Card** for each major vendor
- [ ] Set status to `Active` and verify `valid_from` / `valid_to` dates

---

## 9. Users & Roles

In ERPNext: **Settings → User → New User**

Assign one or more TMS roles per user:

| Role | Typical user |
|---|---|
| Order Desk | Customer service coordinators |
| Dispatcher | Operations / dispatch team |
| Transport Finance | Accountants, billing staff |
| Transport Manager | Operations managers, directors |

!!! note
    Users with `System Manager` role automatically have full access to all TMS doctypes. Avoid assigning System Manager to operational users.

---

## 10. Verify with a Test Order

Run the [Quick Start](getting-started.md) walkthrough using a test customer. Confirm:

- [ ] Rate card auto-fills on order items
- [ ] Trip creates successfully with trailer availability check
- [ ] Trip costs post to correct GL accounts
- [ ] Sales Invoice generates with correct VAT
- [ ] Dashboard KPI cards update
