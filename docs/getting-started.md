# Quick Start

**Audience:** All users. This walkthrough takes you from first login to a completed Transport Order → Trip → Invoice in about 15 minutes.

---

## Step 1 — Seed Master Data

Before creating orders you need at minimum:

1. **Two Transport Locations** — e.g. "Riyadh Industrial City" and "Jeddah Port"
2. **One Transport Route** — linking those two locations (route name auto-fills as `Riyadh Industrial City → Jeddah Port`)
3. **One Trailer Type** — e.g. "Flatbed 40T"
4. **One Trailer** — plate number, type, ownership = `Owned`
5. **One Driver** — linked to the trailer
6. **One Customer Rate Card** — covering that route + trailer type with a base rate

!!! tip "Demo Data"
    The app ships with a `setup_demo_data` script. Run it to seed BK Transport Co. with locations, routes, rate cards, a tractor, a trailer, and a driver in one command:
    ```bash
    bench --site tms.yourdomain.com execute transport_management.setup.demo_data.seed
    ```

---

## Step 2 — Create a Transport Order

1. Go to **Transport Operations → Transport Order → New**
2. Select **Customer** (e.g. "Arabian Logistics Co.")
3. Add one **item line**:
    - Route: `Riyadh Industrial City → Jeddah Port`
    - Trailer Type: `Flatbed 40T`
    - Qty Trailers: `1`
    - Total Stops: `2` (1 pickup + 1 drop — the minimum)
4. Watch the pricing auto-fill from the rate card: base rate, fuel surcharge, line total
5. Add **stops** in the Stops child table: Pickup at Riyadh Industrial City, Drop at Jeddah Port
6. Click **Save**, then **Submit**

The order status moves to `Confirmed`. A **Create Trip** button appears.

---

## Step 3 — Create and Dispatch the Trip

1. Click **Create Trip** on the submitted Transport Order
2. The Trip form opens pre-filled with the customer, route, and stops
3. Select:
    - **Trailer**: your Flatbed trailer
    - **Driver**: your driver
4. If owned fleet: enter estimated fuel cost under **Trip Costs** (fuel, driver pay, tolls)
5. Click **Save**, then **Submit**
6. Change status to **Dispatched**

The trailer status updates automatically to `On Trip`.

---

## Step 4 — Record Proof of Delivery

1. Open the Trip form
2. In the **Stops** table, find the Drop stop
3. Set stop status to `Completed` and attach a POD document (photo or scan)
4. Once all stops are `Completed`, the Trip status can advance to `Delivered`
5. Submit the POD — the trip status moves to `Completed`

---

## Step 5 — Generate Invoice

1. From the completed Trip (or its parent Transport Order), click **Create Sales Invoice**
2. The invoice pre-fills with line items: Freight Linehaul, Fuel Surcharge (and any extras)
3. VAT (15%) applies automatically via the KSA tax template
4. Review and **Submit** the invoice

For outsourced trips, a **Create Purchase Invoice** button also appears for the subcontractor cost.

---

## What Just Happened

```
Transport Order (TO-2026-00001)
  └── Trip (TRIP-2026-00001)
        ├── Trip Costs: fuel SAR 450, driver SAR 300
        ├── POD: attached at drop stop
        └── Sales Invoice (SINV-2026-00001) — SAR 2,875 incl. VAT
              └── Purchase Invoice (PINV-2026-00001) — SAR 1,200 (if outsourced)
```

Trip profit = selling total − buying total, computed on the Trip form in real time.

---

## Next Steps

| Topic | Link |
|---|---|
| Setting up rate cards | [Rate Cards](rate-cards.md) |
| Order Desk step-by-step | [Order Desk Guide](user-guides/order-desk.md) |
| Dispatch and trip management | [Dispatcher Guide](user-guides/dispatcher.md) |
| Invoicing and reports | [Finance Guide](user-guides/finance.md) |
