# API Reference

**Audience:** Frappe developers building integrations or extending Enigma TMS.

---

## Python Modules

### `transport_pricing/pricing.py`

Core rate resolution and calculation logic. All pricing is server-side — never client-only.

#### `resolve_customer_rate(customer, route, trailer_type, order_date)`

Resolves the applicable Customer Rate Card Line for a given order context.

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `customer` | `str` | Frappe Customer name |
| `route` | `str` | Transport Route name |
| `trailer_type` | `str` | Trailer Type name |
| `order_date` | `date` | The order date used to check card validity |

**Returns:** `dict` — all fields from the matching `Customer Rate Card Line`

**Raises:** `frappe.ValidationError` if no valid card or line is found

```python
from transport_management.transport_pricing.pricing import resolve_customer_rate

rate = resolve_customer_rate(
    customer="Arabian Logistics Co.",
    route="RT-00001",
    trailer_type="Flatbed 40T",
    order_date=frappe.utils.today()
)
# rate = {"base_rate": 2000, "extra_stop_charge": 300, "fuel_surcharge_pct": 5.0, ...}
```

---

#### `resolve_subcontractor_rate(subcontractor, route, trailer_type, trip_date)`

Same as above but for Subcontractor Rate Cards.

**Parameters:** Same structure, `subcontractor` is a Frappe Supplier name.

**Returns:** `dict` from `Subcontractor Rate Card Line`

---

#### `calculate_line_total(rate_line, total_stops, qty_trailers, detention_hours=0, apply_fuel_on="Linehaul")`

Applies the pricing formula to a resolved rate line.

**Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `rate_line` | `dict` | — | Output of `resolve_customer_rate` or `resolve_subcontractor_rate` |
| `total_stops` | `int` | — | Total stops including pickup + all drops |
| `qty_trailers` | `int` | — | Number of trailers on this leg |
| `detention_hours` | `float` | `0` | Hours of detention recorded |
| `apply_fuel_on` | `str` | `"Linehaul"` | `"Linehaul"` or `"Subtotal"` |

**Returns:** `dict`

```python
{
    "extra_stop_amount": 300.0,
    "detention_amount": 0.0,
    "fuel_surcharge_amount": 100.0,
    "line_total": 2400.0,
}
```

**Example:**

```python
from transport_management.transport_pricing.pricing import (
    resolve_customer_rate, calculate_line_total
)

rate = resolve_customer_rate("Arabian Logistics Co.", "RT-00001", "Flatbed 40T", "2026-03-15")
totals = calculate_line_total(rate, total_stops=3, qty_trailers=2, detention_hours=0)
# totals["line_total"] == max(min_charge, (2000 + 300 + 100) * 2) == 4800
```

---

#### `calculate_trip_profit(trip_doc)`

Updates `selling_amount`, `buying_total`, `trip_profit`, and `trip_margin_pct` on a Trip document in-memory (does not save).

**Parameter:** `trip_doc` — a Frappe Trip document object

```python
trip = frappe.get_doc("Trip", "TRIP-2026-00001")
calculate_trip_profit(trip)
trip.save()
```

---

### `transport_operations/utils.py`

#### `create_sales_invoice_from_order(transport_order_name)`

Creates a draft Sales Invoice from a Transport Order. Returns the new Sales Invoice name.

```python
from transport_management.transport_operations.utils import create_sales_invoice_from_order

sinv_name = create_sales_invoice_from_order("TO-2026-00001")
# sinv_name == "SINV-2026-00001"
```

#### `create_purchase_invoice_from_trip(trip_name)`

Creates a draft Purchase Invoice from an outsourced Trip.

#### `post_trip_costs_to_gl(trip_name)`

Posts owned-fleet Trip Costs as a GL Journal Entry. Called automatically on Trip submit if `auto_post_costs = Yes` in Transport Setting.

---

## Frappe Hooks (`hooks.py`)

Key hooks registered by the app:

```python
# hooks.py

doc_events = {
    "Transport Order": {
        "validate": "transport_management.transport_operations.transport_order.validate",
        "before_submit": "transport_management.transport_operations.transport_order.before_submit",
        "on_submit": "transport_management.transport_operations.transport_order.on_submit",
        "on_cancel": "transport_management.transport_operations.transport_order.on_cancel",
    },
    "Trip": {
        "validate": "transport_management.transport_operations.trip.validate",
        "before_submit": "transport_management.transport_operations.trip.before_submit",
        "on_submit": "transport_management.transport_operations.trip.on_submit",
        "on_cancel": "transport_management.transport_operations.trip.on_cancel",
    },
}

fixtures = [
    "Role",
    "Custom Field",
    "Workspace",
    "Number Card",
    "Dashboard Chart",
    "Report",
]
```

### Key Hook Implementations

**`transport_order.validate`:**
- Validates stops table (min 2 stops, at least 1 pickup + 1 drop)
- Resolves rate card and computes item totals
- Checks customer credit limit (if enabled)

**`transport_order.on_submit`:**
- Sets status to `Confirmed`
- Locks pricing fields

**`trip.before_submit`:**
- Checks trailer availability (no overlapping active trips)
- Checks driver availability
- Resolves subcontractor rate if outsourced

**`trip.on_submit`:**
- Sets trailer status to `On Trip`
- Posts GL Journal Entry for owned costs (if configured)
- Creates draft Sales Invoice (if `auto_create_sales_invoice = Yes`)

**`trip.on_cancel`:**
- Returns trailer to `Available`
- Cancels linked GL Journal Entry

---

## REST API

Enigma TMS exposes its doctypes via the standard Frappe REST API at `/api/resource/`.

### Transport Order

```bash
# List (last 20 confirmed orders)
GET /api/resource/Transport Order?filters=[["status","=","Confirmed"]]&limit=20

# Get single order
GET /api/resource/Transport Order/TO-2026-00001

# Create (Draft)
POST /api/resource/Transport Order
Content-Type: application/json

{
  "customer": "Arabian Logistics Co.",
  "order_date": "2026-03-15",
  "required_by": "2026-03-17",
  "stops": [
    {"seq": 1, "stop_type": "Pickup", "location": "LOC-00001"},
    {"seq": 2, "stop_type": "Drop",   "location": "LOC-00002"}
  ],
  "items": [
    {
      "route": "RT-00001",
      "trailer_type": "Flatbed 40T",
      "qty_trailers": 1,
      "total_stops": 2
    }
  ]
}
```

The server resolves and populates pricing fields on save — do not pass `base_rate`, `line_total`, etc. in API writes.

### Trip

```bash
# Submit a trip
PUT /api/resource/Trip/TRIP-2026-00001
{"docstatus": 1}

# Update stop status (attach POD)
PUT /api/resource/Trip/TRIP-2026-00001
{
  "stops": [
    {"name": "row-uuid", "status": "Completed", "actual_datetime": "2026-03-16 14:30:00"}
  ]
}
```

### Whitelisted Methods

Custom server methods exposed via `@frappe.whitelist()`:

```bash
# Resolve rate for a given context (useful for price preview)
POST /api/method/transport_management.transport_pricing.pricing.get_rate_preview
{
  "customer": "Arabian Logistics Co.",
  "route": "RT-00001",
  "trailer_type": "Flatbed 40T",
  "total_stops": 3,
  "qty_trailers": 1
}
```

---

## Extending the Schema

To add a field to Transport Order without modifying app code:

```bash
bench --site your-site.com execute frappe.custom.doctype.custom_field.custom_field.create_custom_field \
  --args '{"dt": "Transport Order", "fieldname": "special_instructions", "fieldtype": "Small Text", "label": "Special Instructions"}'
```

Or use the ERPNext UI: **Settings → Custom Fields → New**.

Custom fields added this way survive app upgrades. Do not edit the app's base JSON directly.

---

## Scheduler Events

```python
# hooks.py
scheduler_events = {
    "daily": [
        "transport_management.tasks.expire_rate_cards",   # sets Status=Expired for past valid_to
        "transport_management.tasks.warn_license_expiry", # emails manager if driver license < 30 days
    ],
    "hourly": [
        "transport_management.tasks.check_overdue_trips", # flags trips past planned_end
    ],
}
```

---

## Testing

Unit tests live in `transport_management/tests/`. Run with:

```bash
bench --site your-site.com run-tests --app transport_management
```

### Writing Rate Card Tests

```python
# tests/test_pricing.py
import frappe
import unittest
from transport_management.transport_pricing.pricing import calculate_line_total

class TestPricing(unittest.TestCase):
    def test_extra_stops(self):
        rate = {
            "base_rate": 2000, "extra_stop_charge": 300,
            "detention_free_hours": 4, "detention_rate_per_hour": 150,
            "fuel_surcharge_pct": 5.0, "min_charge": 1500,
        }
        result = calculate_line_total(rate, total_stops=4, qty_trailers=1)
        # extra_stops = 2 → linehaul = 2000 + 600 = 2600
        # fuel = 2600 * 0.05 = 130
        # line_total = max(1500, 2600 + 130) * 1 = 2730
        self.assertEqual(result["line_total"], 2730)

    def test_min_charge_applies(self):
        rate = {
            "base_rate": 500, "extra_stop_charge": 0,
            "detention_free_hours": 0, "detention_rate_per_hour": 0,
            "fuel_surcharge_pct": 0, "min_charge": 1500,
        }
        result = calculate_line_total(rate, total_stops=2, qty_trailers=1)
        self.assertEqual(result["line_total"], 1500)  # min_charge wins
```
