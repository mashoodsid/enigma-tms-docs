# Customization Guide

**Audience:** Frappe developers extending Enigma TMS for specific business needs.

---

## Adding Fields to Core Doctypes

Use Frappe Custom Fields — never edit the app's base JSON directly.

### Via UI

**Settings → Custom Fields → New Custom Field**

- DocType: e.g. `Transport Order`
- Label + Fieldname
- Fieldtype (Data, Select, Currency, etc.)
- Insert After: choose where it appears in the form

### Via Script (Reproducible)

```python
frappe.get_doc({
    "doctype": "Custom Field",
    "dt": "Transport Order",
    "fieldname": "special_instructions",
    "fieldtype": "Small Text",
    "label": "Special Instructions",
    "insert_after": "customer_po_no",
    "in_list_view": 0,
}).insert()
```

Add this to your add-on app's `after_install` hook so it runs on every fresh install.

---

## Adding a New Trip Status

The Trip status field is a Select. To add a new state (e.g. `Customs Hold` for cross-border trips):

1. **Custom Field** → Edit the `Trip.status` Select field → add `Customs Hold` to the options list
2. **Workflow** → update the Frappe Workflow on Trip to allow transitions into/out of `Customs Hold`
3. **Client JS** → add any client-side warnings or hide/show logic for the new status

!!! warning
    The double-booking check and POD gate in `trip.before_submit` and `trip.on_update_after_submit` reference specific status values. Review those functions in `transport_operations/trip.py` before adding new statuses to ensure they behave correctly.

---

## Adding a Custom Report

Frappe Query Reports are the recommended approach for custom analytics.

```python
# your_app/report/my_custom_report/my_custom_report.py

def execute(filters=None):
    columns = [
        {"label": "Trip", "fieldname": "name", "fieldtype": "Link",
         "options": "Trip", "width": 120},
        {"label": "Customer", "fieldname": "customer", "fieldtype": "Data", "width": 150},
        {"label": "Route", "fieldname": "route", "fieldtype": "Data", "width": 200},
        {"label": "Profit (SAR)", "fieldname": "trip_profit",
         "fieldtype": "Currency", "width": 120},
    ]

    data = frappe.db.sql("""
        SELECT
            t.name,
            t.customer,
            r.route_name as route,
            t.trip_profit
        FROM `tabTrip` t
        LEFT JOIN `tabTransport Route` r ON t.route = r.name
        WHERE t.docstatus = 1
          AND t.status = 'Completed'
          AND t.actual_end BETWEEN %(from_date)s AND %(to_date)s
        ORDER BY t.trip_profit ASC
    """, filters, as_dict=True)

    return columns, data
```

Add the report JSON to your app's `fixtures` to make it appear in the Reports menu.

---

## Extending the Pricing Engine

To add a new pricing component (e.g. a weekend surcharge):

1. Add a `weekend_surcharge_pct` field to both `Customer Rate Card Line` and `Subcontractor Rate Card Line` via Custom Fields
2. Override `calculate_line_total` in your add-on:

```python
# your_app/pricing_extension.py
from transport_management.transport_pricing.pricing import calculate_line_total as _base_calc
import frappe

def calculate_line_total(rate_line, total_stops, qty_trailers,
                         detention_hours=0, apply_fuel_on="Linehaul",
                         order_date=None):
    result = _base_calc(rate_line, total_stops, qty_trailers,
                        detention_hours, apply_fuel_on)

    if order_date and frappe.utils.getdate(order_date).weekday() >= 4:  # Fri/Sat in KSA
        weekend_pct = rate_line.get("weekend_surcharge_pct", 0)
        surcharge = result["line_total"] * weekend_pct / 100
        result["weekend_surcharge_amount"] = surcharge
        result["line_total"] += surcharge

    return result
```

3. Hook into `Transport Order` validate to call your extended function instead

---

## Custom Workspace Shortcuts

To add a quick link to the Transport Operations workspace:

**Settings → Workspace → Transport Operations → Edit → Add Shortcut**

Or add via fixture:

```json
{
    "doctype": "Workspace Shortcut",
    "parent": "Transport Operations",
    "label": "My Custom Report",
    "type": "Report",
    "link_to": "My Custom Report",
    "color": "#2490EF"
}
```

---

## Print Formats

The app ships with a **Trip Sheet** print format (for the driver to carry) and a **Transport Order Confirmation** format.

To customize them: **Settings → Print Format → [select] → Edit**

The Trip Sheet uses Jinja templating:

```html
<!-- Key variables available in the template -->
{{ doc.name }}
{{ doc.customer }}
{% for stop in doc.stops %}
  {{ stop.seq }}. {{ stop.stop_type }}: {{ stop.location }}
{% endfor %}
```
