# Order Desk Guide

**Role:** Order Desk / Coordinator
**Can:** Create and submit Transport Orders, view rate cards and routes
**Cannot:** Create Trips, generate invoices, modify rate cards

---

## Daily Workflow

Your primary job is to convert customer requests into confirmed, priced Transport Orders that the Dispatcher can act on.

```
Customer calls/emails  →  Create Transport Order  →  Submit  →  Dispatcher takes over
```

---

## Creating a Transport Order

### 1. Open a New Order

Navigate to **Transport Operations → Transport Order → New** (or press `Alt+N` from the list view).

### 2. Fill the Header

| Field | What to enter |
|---|---|
| **Customer** | Start typing the customer name; select from the list |
| **Customer PO No.** | Their purchase order or reference number (optional but useful for disputes) |
| **Order Date** | Today's date (auto-filled) |
| **Required By** | The customer's requested delivery date |
| **Currency** | SAR (default; change only for foreign-currency contracts) |

### 3. Add Stops

In the **Stops** table, add each location in execution order:

| Seq | Stop Type | Location |
|---|---|---|
| 1 | Pickup | Riyadh Industrial City |
| 2 | Drop | Jeddah Port |

For multi-stop orders:

| Seq | Stop Type | Location |
|---|---|---|
| 1 | Pickup | Riyadh Industrial City |
| 2 | Drop | Yanbu Factory |
| 3 | Drop | Jeddah Port |

Fill `Planned Datetime` if the customer has a specific slot — the Dispatcher and Driver will see this.

### 4. Add Order Items (Priced Lines)

In the **Items** table, click **Add Row** for each route segment:

| Field | What to enter |
|---|---|
| **Route** | Select the Transport Route (e.g. `Riyadh IC → Jeddah Port`) |
| **Trailer Type** | Select the required trailer type (e.g. `Flatbed 40T`) |
| **Qty Trailers** | Number of trailers for this line |
| **Total Stops** | Total stops on this leg (minimum 2: 1 pickup + 1 drop) |

After you select the Route and Trailer Type, pricing auto-fills from the customer's active rate card:
- Base Rate
- Extra Stop Charge (if Total Stops > 2)
- Fuel Surcharge %
- Min Charge
- **Line Total** (computed)

!!! warning "No Rate Card Found"
    If you see an error "No active Customer Rate Card found", the rate for this route/trailer type hasn't been set up yet. Contact Transport Finance to create the rate card before proceeding.

### 5. Review Pricing

Check the totals at the bottom:
- **Total** — net of VAT
- **Total Taxes** — 15% VAT
- **Grand Total** — what the customer owes

If the computed price differs from what was quoted to the customer, **do not manually override** line totals. Contact Finance to update the rate card, then refresh the order.

### 6. Save and Submit

- **Save** — keeps the order in `Draft` status; you can still edit
- **Submit** — locks the order (status → `Confirmed`); pricing is frozen; the Dispatcher can now create a Trip

!!! note "Credit Check"
    If the customer has exceeded their credit limit, you'll see a warning or hard block at submission (depending on Transport Setting). Contact Finance to resolve before proceeding.

---

## Amending a Submitted Order

Once submitted, order items are locked. To change a price or route:

1. Open the Transport Order
2. Click **Cancel** (status must be `Confirmed` with no active Trips)
3. Click **Amend** — a new Draft is created with the same data
4. Make your changes
5. Submit the amended order

The original order is marked `Cancelled`; the amended one gets a new document name with a suffix.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| "No active Customer Rate Card found" | No rate set for this route + trailer type | Ask Finance to create the rate card |
| "Customer credit limit exceeded" | Outstanding balance too high | Ask Finance to approve credit exception |
| "Route from_location = to_location" | Both ends of the route are the same | Check route selection |
| "Total Stops must be at least 2" | Only 1 stop entered | Add both a Pickup and at least one Drop |
| "Stops table is empty" | No stops added | Add stops in the Stops child table |

---

## Tips

- Use the **Customer filter** in the Transport Order list to see all orders for a specific customer
- The **Status** column in the list view shows at a glance what stage each order is at
- Orders in `In Progress` have active Trips — changes require coordination with the Dispatcher
- You can duplicate an existing order (Menu → Duplicate) for repeat customers with identical lanes
