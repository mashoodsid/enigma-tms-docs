# Hooks & Events

**Audience:** Developers extending or integrating with Enigma TMS.

This page documents all Frappe hooks registered by `transport_management` and the events you can hook into from your own app.

---

## Document Events

### Transport Order

| Hook | Function | What it does |
|---|---|---|
| `validate` | `transport_order.validate` | Validates stops, resolves pricing, credit check |
| `before_submit` | `transport_order.before_submit` | Final validation before submit |
| `on_submit` | `transport_order.on_submit` | Sets status `Confirmed`, locks pricing fields |
| `on_cancel` | `transport_order.on_cancel` | Sets status `Cancelled`, cancels linked trips |
| `on_update_after_submit` | `transport_order.on_update_after_submit` | Handles status progression (In Progress → Completed) |

### Trip

| Hook | Function | What it does |
|---|---|---|
| `validate` | `trip.validate` | Computes trip profit, validates stop sequence |
| `before_submit` | `trip.before_submit` | Trailer/driver availability check, rate resolution |
| `on_submit` | `trip.on_submit` | Sets trailer `On Trip`, posts GL, maybe creates invoice |
| `on_cancel` | `trip.on_cancel` | Returns trailer to `Available`, reverses GL entry |
| `on_update_after_submit` | `trip.on_update_after_submit` | Handles stop status changes, POD validation |

---

## Scheduler Events

| Frequency | Function | Purpose |
|---|---|---|
| Daily | `tasks.expire_rate_cards` | Auto-expire cards past `valid_to` |
| Daily | `tasks.warn_license_expiry` | Email Transport Manager for drivers with license expiring in ≤ 30 days |
| Hourly | `tasks.check_overdue_trips` | Flag trips past `planned_end` on the dashboard |

---

## Fixtures

Installed via `bench --site migrate`:

```python
fixtures = [
    "Role",                 # Order Desk, Dispatcher, Transport Finance, Transport Manager
    "Custom Field",         # driver_type, national_id_iqama, license_expiry on Driver doctype
    "Workspace",            # Transport Operations workspace
    "Number Card",          # 11 KPI cards
    "Dashboard Chart",      # 9 charts
    "Report",               # 6 business reports
    "Transport Setting",    # default singleton values
]
```

---

## Hooking Into TMS Events from Your App

If you build an add-on (e.g. a GPS telematics module), hook into TMS events in your own `hooks.py`:

```python
# your_app/hooks.py
doc_events = {
    "Trip": {
        "on_submit": "your_app.telematics.on_trip_submit",
        "on_cancel": "your_app.telematics.on_trip_cancel",
    }
}
```

Your handler receives the document object:

```python
# your_app/telematics.py
import frappe

def on_trip_submit(doc, method):
    if doc.ownership == "Owned":
        # Start GPS tracking for this trailer
        frappe.enqueue(
            "your_app.telematics.start_tracking",
            trip=doc.name,
            trailer=doc.trailer,
        )
```

---

## Client-Side Events

Key form events in `transport_management/public/js/`:

### `transport_order.js`

| Event | Trigger | Action |
|---|---|---|
| `onload` | Form opens | Set up dynamic filters on Route/Trailer Type fields |
| `items.route` / `items.trailer_type` change | User selects route or type | Fetch and populate rate card fields via `frappe.call` |
| `items.total_stops` change | Stops count updated | Recalculate extra stop amount client-side (server recalculates on save) |
| `before_submit` | User clicks Submit | Client-side validation: at least 1 pickup + 1 drop in stops table |

### `trip.js`

| Event | Trigger | Action |
|---|---|---|
| `trailer` change | Trailer selected | Auto-fill ownership, subcontractor; fetch default driver |
| `ownership` change | Ownership changes | Show/hide subcontractor field; show/hide costs table |
| `status` change | Status updated | Warn if required PODs are missing before allowing Delivered |
