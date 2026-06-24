# Dispatcher Guide

**Role:** Dispatcher / Operations
**Can:** Create and manage Trips, assign assets and drivers, update stop status, collect POD
**Cannot:** Create Transport Orders, generate invoices, modify rate cards

---

## The Dispatch Board

Your primary workspace is the **Kanban view** on Trip, accessible from:

**Transport Operations → Trip → (switch to Kanban view)**

Columns map to Trip status: `Assigned | Dispatched | At Pickup | Loaded | In Transit | At Drop | Delivered`

From the Kanban board you can:
- See at a glance which trips are waiting to be dispatched, in transit, or completed
- Open a Trip card to assign assets or update status
- Filter by date range using the standard Frappe filters

---

## Creating a Trip

Trips are created from a submitted Transport Order.

1. Open the Transport Order (status must be `Confirmed`)
2. Click **Create Trip**
3. The Trip form opens pre-filled:
    - Customer and stops copied from the Order
    - `selling_amount` pulled from the Order's selling total

### For Owned Fleet

| Field | Action |
|---|---|
| **Trailer** | Select an `Available` trailer of the right type |
| **Tractor** | Select the prime mover (optional) |
| **Driver** | Select the driver — verify license is current |
| **Planned Start / End** | Set the expected departure and arrival times |

The system will block you if the trailer or driver is already assigned to another overlapping trip.

### For Outsourced (Subcontracted) Trips

| Field | Action |
|---|---|
| **Trailer** | Select the outsourced trailer (ownership = Outsourced) |
| **Driver** | Select a subcontractor driver or leave blank if vendor supplies |
| **Subcontractor** | Auto-filled from the trailer; verify it's the right vendor |

The buying cost (`subcon_line_total`) is resolved automatically from the Subcontractor Rate Card.

### Entering Owned-Fleet Costs

In the **Trip Costs** table, log each cost:

| Cost Type | Example Amount | Description |
|---|---|---|
| Fuel | SAR 450 | Fuel fill-up at Riyadh depot |
| Driver Pay | SAR 300 | Per-trip driver allowance |
| Toll | SAR 75 | Highway toll gates |
| Other | SAR 50 | Miscellaneous |

These costs are used to calculate the trip profit and will be posted to GL when the trip is completed.

### Submit the Trip

Click **Submit** to activate the trip. This:
- Moves the Trailer status to `On Trip`
- Sets the Trip status to `Assigned`
- Locks the trailer and driver from other overlapping assignments

---

## Advancing Trip Status

Move the trip through its lifecycle by updating the **Status** field:

```
Assigned → Dispatched → At Pickup → Loaded → In Transit → At Drop → Delivered → Completed
```

Each status change is logged with a timestamp. Key points:

- **Dispatched** — when the truck leaves the depot
- **At Pickup** — when the driver arrives at the pickup location
- **Loaded** — cargo on board; weight/seal confirmed
- **In Transit** — truck en route
- **At Drop** — driver at drop location
- **Delivered** — cargo handed over; POD required before this → `Completed`

---

## Proof of Delivery (POD)

POD is **required** before a stop can be marked `Completed`, and all stops must be `Completed` before the Trip can reach `Completed`.

For each **Drop stop** in the Stops table:

1. Set `Status` → `Arrived`
2. Set `Actual Datetime` to the arrival time
3. Attach the POD document (photo, signed delivery note, scan) in `POD Attachment`
4. Set `Status` → `Completed`

Once all stops are `Completed`:
- Change Trip Status to `Delivered`, then `Completed`
- If `auto_create_sales_invoice` is enabled, a draft Sales Invoice is created automatically
- The Trailer status returns to `Available`

---

## Handling Deviations

### Driver Cannot Complete the Trip

1. Do **not** cancel the trip yet — contact the Transport Manager
2. Update the trip notes with the incident details
3. If reassigning: change the Driver/Trailer and re-submit (requires cancelling and amending)

### Route Change Mid-Trip

If the actual route differs from the order:
1. Note it in trip comments
2. Flag the order for repricing — Finance will create a credit/debit note
3. Update stop locations in the Stops table (only allowed while trip is not yet Completed)

### Cost Overrun

Add the actual cost in Trip Costs with the correct type and a note in the description. The trip profit will reflect the true margin. Finance reviews margin outliers in the Trip Profitability report.

### POD Refused or Missing

If a consignee refuses to sign:
1. Attach a photo of the cargo at the drop point as evidence
2. Add a note in the Stop's `Load Description` field
3. Escalate to Order Desk to contact the customer
4. Do not mark the stop `Completed` without some form of proof

---

## Trailer Availability

To check which trailers are free:

- Go to **Transport Setup → Trailer**
- Filter by `Status = Available` and `Trailer Type`
- The list shows all available units of the required type

---

## Key Reports for Dispatchers

| Report | Where | Purpose |
|---|---|---|
| Active Trips | Dashboard KPI card | Count of trips currently in progress |
| Trailers Available | Dashboard KPI card | Fleet availability right now |
| Trip Profitability | Reports menu | Margin per trip; spot cost overruns |
| Fleet Utilization | Reports menu | How many days each trailer was on trip |
