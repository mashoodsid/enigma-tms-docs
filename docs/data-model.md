# Data Model

**Audience:** Developers building integrations or extending the schema.

This page covers field-level detail for every doctype. For relationships and flows see [System Overview](architecture.md).

---

## Tractor

Naming: `TRC-.#####`

| Field | Type | Required | Notes |
|---|---|---|---|
| `plate_number` | Data | Yes | Unique; the prime mover registration |
| `ownership` | Select | Yes | `Owned` / `Outsourced` |
| `subcontractor` | Link → Supplier | If Outsourced | `mandatory_depends_on: eval:doc.ownership=='Outsourced'` |
| `make_model` | Data | — | |
| `model_year` | Int | — | |
| `status` | Select | — | `Available` / `On Trip` / `Under Maintenance` / `Inactive`; system-maintained |
| `company` | Link → Company | — | |

---

## Trailer

Naming: `TRL-.#####`

| Field | Type | Required | Notes |
|---|---|---|---|
| `plate_number` | Data | Yes | Unique |
| `trailer_type` | Link → Trailer Type | Yes | |
| `ownership` | Select | Yes | `Owned` / `Outsourced` |
| `subcontractor` | Link → Supplier | If Outsourced | |
| `capacity_tons` | Float | — | |
| `capacity_cbm` | Float | — | Volume capacity |
| `make_model` | Data | — | |
| `model_year` | Int | — | |
| `status` | Select | — | `Available` / `On Trip` / `Under Maintenance` / `Inactive` |
| `default_driver` | Link → Driver | — | Optional default; can be overridden per trip |
| `company` | Link → Company | — | |

---

## Transport Location

Naming: `LOC-.#####`

| Field | Type | Required | Notes |
|---|---|---|---|
| `location_name` | Data | Yes | |
| `location_type` | Select | — | `Customer Site` / `Warehouse` / `Factory` / `Port` / `Border Crossing` / `Other` |
| `city` | Link → City | — | |
| `country` | Link → Country | — | Default: Saudi Arabia |
| `geolocation` | Geolocation | — | Map pin; used by live fleet map |
| `address` | Link → Address | — | Reuses ERPNext Address |
| `contact_person` | Data | — | |
| `contact_phone` | Data | — | |

---

## Transport Route

Naming: `RT-.#####` — `route_name` auto-set to `{from_location} → {to_location}`

| Field | Type | Required | Notes |
|---|---|---|---|
| `from_location` | Link → Transport Location | Yes | |
| `to_location` | Link → Transport Location | Yes | |
| `from_city` | Link → City | — | Fetched from location |
| `to_city` | Link → City | — | Fetched from location |
| `from_country` | Link → Country | — | Fetched |
| `to_country` | Link → Country | — | Fetched |
| `is_cross_border` | Check | — | Auto: `from_country != to_country` |
| `distance_km` | Float | — | Auto-filled by maps API if enabled |
| `standard_transit_hours` | Float | — | Auto-filled by maps API |

---

## Customer Rate Card

Naming: `CRC-.YYYY.-.#####`

| Field | Type | Notes |
|---|---|---|
| `customer` | Link → Customer | Blank = default card for all customers |
| `valid_from` | Date | |
| `valid_to` | Date | |
| `currency` | Link → Currency | Default SAR |
| `status` | Select | `Draft` / `Active` / `Expired` |
| `lines` | Table → Customer Rate Card Line | Rate lines per route + trailer type |

### Customer Rate Card Line (Child)

| Field | Type | Notes |
|---|---|---|
| `route` | Link → Transport Route | |
| `trailer_type` | Link → Trailer Type | |
| `base_rate` | Currency | Linehaul for 1 pickup + 1 drop |
| `extra_stop_charge` | Currency | Per additional stop beyond 2 |
| `detention_free_hours` | Float | Grace period before detention billing starts |
| `detention_rate_per_hour` | Currency | |
| `fuel_surcharge_pct` | Percent | Applied to linehaul (configurable) |
| `min_charge` | Currency | Floor amount per trailer |

---

## Subcontractor Rate Card

Naming: `SRC-.YYYY.-.#####` — same structure as Customer Rate Card but keyed by `subcontractor` (Link → Supplier) and cost field names use `_cost` suffix: `base_cost`, `extra_stop_cost`, `detention_cost_per_hour`, `min_cost`.

---

## Transport Order

Naming: `TO-.YYYY.-.#####`

| Field | Type | Notes |
|---|---|---|
| `customer` | Link → Customer | |
| `customer_po_no` | Data | Optional customer reference |
| `order_date` | Date | |
| `required_by` | Date | Requested delivery date |
| `currency` | Link → Currency | Default SAR |
| `stops` | Table → Transport Stop | Multi-stop plan |
| `items` | Table → Transport Order Item | Priced lines |
| `taxes_and_charges` | Link → Sales Taxes Template | KSA VAT 15% |
| `total` | Currency | Net of tax; server-computed |
| `total_taxes` | Currency | Server-computed |
| `grand_total` | Currency | Incl. VAT; server-computed |
| `selling_total` | Currency | Net for margin calc |
| `status` | Select | `Draft` / `Confirmed` / `In Progress` / `Completed` / `Closed` / `Cancelled` |

### Transport Order Item (Child)

| Field | Type | Notes |
|---|---|---|
| `route` | Link → Transport Route | |
| `trailer_type` | Link → Trailer Type | |
| `qty_trailers` | Int | |
| `total_stops` | Int | Incl. pickup + all drops |
| `base_rate` | Currency | Auto-fetched from rate card |
| `extra_stop_charge` | Currency | Auto-fetched |
| `extra_stop_amount` | Currency | Computed |
| `detention_hours` | Float | Entered manually or by system |
| `detention_amount` | Currency | Computed |
| `fuel_surcharge_pct` | Percent | Auto-fetched |
| `fuel_surcharge_amount` | Currency | Computed |
| `min_charge` | Currency | Auto-fetched |
| `line_total` | Currency | Final line amount |

---

## Transport Stop (Child — shared by Order and Trip)

| Field | Type | Notes |
|---|---|---|
| `seq` | Int | Execution sequence |
| `stop_type` | Select | `Pickup` / `Drop` |
| `location` | Link → Transport Location | |
| `contact_person` | Data | |
| `contact_phone` | Data | |
| `planned_datetime` | Datetime | |
| `load_description` | Small Text | |
| `weight_tons` | Float | |
| `status` | Select | `Pending` / `Arrived` / `Completed` (updated during trip execution) |
| `actual_datetime` | Datetime | Filled when stop is actioned |
| `pod_attachment` | Attach | POD document (required before stop = Completed) |

---

## Trip

Naming: `TRIP-.YYYY.-.#####`

| Field | Type | Notes |
|---|---|---|
| `transport_order` | Link → Transport Order | |
| `customer` | Link → Customer | Fetched |
| `trailer` | Link → Trailer | |
| `tractor` | Link → Tractor | Optional; for prime-mover tracking |
| `ownership` | Select | Fetched from Trailer |
| `subcontractor` | Link → Supplier | Fetched if Outsourced |
| `driver` | Link → Driver | |
| `planned_start` | Datetime | |
| `actual_start` | Datetime | |
| `planned_end` | Datetime | |
| `actual_end` | Datetime | |
| `odometer_start` | Float | |
| `odometer_end` | Float | |
| `actual_distance` | Float | Computed: end − start |
| `stops` | Table → Transport Stop | Copied from order; updated live |
| `costs` | Table → Trip Costs | Owned-fleet cost breakdown |
| `subcon_line_total` | Currency | Resolved from Subcontractor Rate Card |
| `selling_amount` | Currency | From parent order |
| `buying_total` | Currency | Sum of costs or subcon total |
| `trip_profit` | Currency | Selling − buying |
| `trip_margin_pct` | Percent | |
| `status` | Select | `Assigned` → `Dispatched` → ... → `Completed` |

### Trip Costs (Child)

| Field | Type | Notes |
|---|---|---|
| `cost_type` | Select | `Fuel` / `Driver Pay` / `Toll` / `Other` |
| `amount` | Currency | |
| `description` | Data | |
| `gl_account` | Link → Account | Optional override for GL posting |
