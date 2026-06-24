# System Overview

**Audience:** Frappe developers and technical architects. Assumes familiarity with Frappe doctypes, modules, and the ERPNext data model.

---

## Module Structure

Enigma TMS is a single Frappe app (`transport_management`) organized into three modules:

```
transport_management/
├── transport_setup/          # Masters: assets, locations, routes
│   ├── doctype/
│   │   ├── tractor/
│   │   ├── trailer/
│   │   ├── trailer_type/
│   │   ├── transport_location/
│   │   └── transport_route/
│
├── transport_pricing/        # Rate card engine
│   ├── doctype/
│   │   ├── customer_rate_card/
│   │   ├── customer_rate_card_line/
│   │   ├── subcontractor_rate_card/
│   │   └── subcontractor_rate_card_line/
│   └── pricing.py            # Core pricing logic
│
└── transport_operations/     # Transactions
    ├── doctype/
    │   ├── transport_order/
    │   ├── transport_order_item/
    │   ├── transport_stop/
    │   ├── trip/
    │   └── trip_costs/
    └── utils.py
```

ERPNext doctypes used (not owned by this app): `Customer`, `Supplier`, `Driver`, `Sales Invoice`, `Purchase Invoice`, `Journal Entry`, `Item`, `Company`.

---

## Data Flow

```
                    ┌─────────────────────────────────────────────┐
                    │              MASTER DATA                      │
   Customer ────►  Customer Rate Card   Subcontractor Rate Card ◄─ Supplier
   Transport Location ────► Transport Route
   Trailer Type ────► Trailer ◄──── Tractor    Driver
                    └─────────────────────────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │            TRANSPORT ORDER                   │
                    │  (submittable, multi-lane, priced from RC)  │
                    │   items[] ← pricing.resolve_customer_rate() │
                    └──────────────────┬──────────────────────────┘
                                       │  Create Trip
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │                 TRIP                         │
                    │  (submittable, execution + costing)         │
                    │  stops[] ← copied from Order               │
                    │  costs[] ← fuel / driver / tolls           │
                    │  buying ← pricing.resolve_subcon_rate()    │
                    │  profit = selling − buying                  │
                    └──────────┬───────────────┬──────────────────┘
                               │               │
                    POD gate   ▼               ▼
              ┌──────────────────┐   ┌──────────────────┐
              │  Sales Invoice   │   │ Purchase Invoice  │
              │  (to customer,   │   │ (to subcontractor│
              │   15% VAT)       │   │  if outsourced)  │
              └──────────────────┘   └──────────────────┘
                               │
                               ▼
                    ┌──────────────────┐
                    │  Journal Entry   │
                    │  (owned-fleet    │
                    │   trip costs)    │
                    └──────────────────┘
```

---

## Doctype Reference

### Master Doctypes

| Doctype | Naming | Key Fields | Submittable |
|---|---|---|---|
| `Tractor` | `TRC-.#####` | plate_number, ownership, subcontractor, status | No |
| `Trailer` | `TRL-.#####` | plate_number, trailer_type, ownership, subcontractor, capacity_tons, status | No |
| `Trailer Type` | `field:type_name` | type_name, default_capacity_tons | No |
| `Transport Location` | `LOC-.#####` | location_name, location_type, city, country, geolocation | No |
| `Transport Route` | `RT-.#####` | from_location, to_location, is_cross_border, distance_km | No |

### Pricing Doctypes

| Doctype | Naming | Key Fields | Submittable |
|---|---|---|---|
| `Customer Rate Card` | `CRC-.YYYY.-.#####` | customer, valid_from, valid_to, currency, status, lines[] | No |
| `Customer Rate Card Line` | (child) | route, trailer_type, base_rate, extra_stop_charge, fuel_surcharge_pct, min_charge | — |
| `Subcontractor Rate Card` | `SRC-.YYYY.-.#####` | subcontractor, valid_from, valid_to, currency, status, lines[] | No |
| `Subcontractor Rate Card Line` | (child) | route, trailer_type, base_cost, extra_stop_cost, fuel_surcharge_pct, min_cost | — |

### Transaction Doctypes

| Doctype | Naming | Key Fields | Submittable |
|---|---|---|---|
| `Transport Order` | `TO-.YYYY.-.#####` | customer, order_date, stops[], items[], total, grand_total, status | **Yes** |
| `Transport Order Item` | (child) | route, trailer_type, qty_trailers, total_stops, line_total | — |
| `Transport Stop` | (child, shared) | seq, stop_type, location, planned_datetime, status, actual_datetime, pod_attachment | — |
| `Trip` | `TRIP-.YYYY.-.#####` | transport_order, trailer, driver, ownership, stops[], costs[], selling_amount, trip_profit, status | **Yes** |
| `Trip Costs` | (child) | cost_type (Fuel/Driver Pay/Toll/Other), amount, description | — |

### Single Doctype

| Doctype | Purpose |
|---|---|
| `Transport Setting` | Global toggles: VAT template, GL accounts, auto-invoice, credit check |

---

## Status State Machines

### Transport Order

```
Draft → Confirmed → In Progress → Completed → Closed
                                      ↓
                                  Cancelled  (from Draft or Confirmed only)
```

| Transition | Trigger |
|---|---|
| `Draft → Confirmed` | Submit the order; rate card validation passes; credit check passes |
| `Confirmed → In Progress` | First Trip is created and submitted |
| `In Progress → Completed` | All Trips reach `Completed` |
| `Completed → Closed` | Sales Invoice submitted (or manual close by Finance) |

### Trip

```
Assigned → Dispatched → At Pickup → Loaded → In Transit → At Drop → Delivered → Completed
```

Each transition is validated server-side. Key gates:

- `Delivered → Completed` requires all stops to have `status = Completed` and a POD attachment
- Submitting a Trip changes the Trailer's `status` to `On Trip`
- Cancelling a Trip returns the Trailer to `Available`

---

## Double-Booking Prevention

Before a Trip can be submitted, the server checks:

```python
# trailer availability
overlapping_trips = frappe.get_all("Trip", filters={
    "trailer": trip.trailer,
    "status": ["not in", ["Completed", "Cancelled"]],
    "name": ["!=", trip.name],
    "planned_start": ["<", trip.planned_end],
    "planned_end": [">", trip.planned_start],
})
if overlapping_trips:
    frappe.throw("Trailer already assigned to {0}".format(overlapping_trips[0].name))
```

The same pattern applies to Driver double-booking.

---

## Security Model

| Role | Doctypes | Permissions |
|---|---|---|
| Order Desk | Transport Order, Transport Location, Transport Route | Read, Write, Submit |
| Dispatcher | Trip, Transport Order (read), Trailer (read/write status) | Read, Write, Submit |
| Transport Finance | All doctypes, Sales/Purchase Invoice, Rate Cards | Read, Write, Submit |
| Transport Manager | All doctypes, reports, settings | Full (including delete) |

Custom fields added to the `Driver` core doctype: `driver_type` (Owned/Subcontractor), `national_id_iqama`, `license_expiry`. These are added via `fixtures/custom_field.json` at install time.
