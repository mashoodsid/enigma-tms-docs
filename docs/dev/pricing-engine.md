# Pricing Engine

**Audience:** Developers and finance staff who need to understand or troubleshoot rate resolution.

The pricing engine lives in `transport_pricing/pricing.py`. All math runs **server-side** in `validate()` — client JS may display previews, but totals are always recomputed on save and submit. This prevents API writes from producing $0 margins.

---

## Rate Card Resolution

When an Order Item is saved, the engine resolves the applicable rate in this order:

```python
def resolve_customer_rate(customer, route, trailer_type, order_date):
    """
    Returns the matching Customer Rate Card Line or raises if none found.
    Resolution precedence:
      1. Customer-specific card valid on order_date
      2. Default card (customer = blank) valid on order_date
    """
    filters = {
        "route": route,
        "trailer_type": trailer_type,
        "status": "Active",
    }
    # Try customer-specific first
    card = frappe.get_all("Customer Rate Card", filters={
        "customer": customer,
        "valid_from": ["<=", order_date],
        "valid_to": [">=", order_date],
        "status": "Active",
    }, limit=1)

    if not card:
        # Fall back to default card
        card = frappe.get_all("Customer Rate Card", filters={
            "customer": "",
            "valid_from": ["<=", order_date],
            "valid_to": [">=", order_date],
            "status": "Active",
        }, limit=1)

    if not card:
        frappe.throw(
            f"No active Customer Rate Card found for route {route}, "
            f"trailer type {trailer_type} on {order_date}"
        )

    lines = frappe.get_all("Customer Rate Card Line", filters={
        "parent": card[0].name,
        "route": route,
        "trailer_type": trailer_type,
    }, fields=["*"], limit=1)

    if not lines:
        frappe.throw(
            f"Rate Card {card[0].name} has no line for route {route} / {trailer_type}"
        )

    return lines[0]
```

The same function exists as `resolve_subcontractor_rate(subcontractor, route, trailer_type, trip_date)` for buying costs.

---

## Pricing Formula

Both selling and buying use the identical formula:

```
extra_stops      = max(0, total_stops − 2)
linehaul         = base_rate + (extra_stops × extra_stop_charge)
detention_amount = max(0, detention_hours − detention_free_hours)
                   × detention_rate_per_hour
fuel_surcharge   = linehaul × fuel_surcharge_pct / 100
                   (applied to linehaul; configurable in Transport Setting)
line_total       = max(min_charge,
                       linehaul + detention_amount + fuel_surcharge)
                   × qty_trailers
```

In Python (`pricing.py`):

```python
def calculate_line_total(rate_line, total_stops, qty_trailers,
                         detention_hours=0, apply_fuel_on="Linehaul"):
    extra_stops = max(0, total_stops - 2)
    linehaul = rate_line.base_rate + (extra_stops * rate_line.extra_stop_charge)

    detention_amount = max(
        0,
        (detention_hours - rate_line.detention_free_hours)
        * rate_line.detention_rate_per_hour
    )

    fuel_base = linehaul if apply_fuel_on == "Linehaul" else linehaul + detention_amount
    fuel_surcharge = fuel_base * (rate_line.fuel_surcharge_pct / 100)

    subtotal = linehaul + detention_amount + fuel_surcharge
    line_total = max(rate_line.min_charge, subtotal) * qty_trailers

    return {
        "extra_stop_amount": extra_stops * rate_line.extra_stop_charge * qty_trailers,
        "detention_amount": detention_amount * qty_trailers,
        "fuel_surcharge_amount": fuel_surcharge * qty_trailers,
        "line_total": line_total,
    }
```

---

## Margin Calculation

Per trip:

```python
def calculate_trip_profit(trip):
    selling_amount = frappe.db.get_value(
        "Transport Order", trip.transport_order, "selling_total"
    )
    buying_total = sum(c.amount for c in trip.costs)
    # For outsourced: buying comes from subcontractor rate card
    if trip.ownership == "Outsourced":
        buying_total = trip.subcon_line_total  # resolved at trip creation
    trip.selling_amount = selling_amount
    trip.trip_profit = selling_amount - buying_total
    trip.trip_margin_pct = (trip.trip_profit / selling_amount * 100
                            if selling_amount else 0)
```

---

## Edge Cases

| Scenario | Behaviour |
|---|---|
| No rate card exists for route + trailer type | Hard error at Order save — cannot proceed |
| Rate card exists but expired (`valid_to < order_date`) | Not matched; falls to default card; error if no default |
| Multiple active cards for same customer + route + type | First returned by query wins; use `valid_from` dates to avoid overlap |
| `min_charge` is higher than computed subtotal | `max(min_charge, subtotal)` applies — customer pays the minimum |
| `total_stops = 1` (pickup only, no drop) | Validates against Order minimum-stop rule before save |
| `detention_hours = 0` and `detention_free_hours = 0` | Zero detention; no extra charge |
| `fuel_surcharge_pct = 0` | No surcharge applied; field can be zero |

---

## Pricing Lock

Once a Transport Order is **submitted** (status moves to `Confirmed`), the pricing fields on `Transport Order Item` are set to `read_only` via `allow_on_submit = 0`. Rate changes on the Rate Card after order submission do not affect in-flight orders.

To reprice a submitted order you must **Cancel and Amend** it — standard Frappe amend workflow.
