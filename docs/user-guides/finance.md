# Finance Guide

**Role:** Transport Finance
**Can:** Manage rate cards, generate invoices, post costs, run all reports
**Cannot:** Create/modify Transport Orders or Trips (read-only on operations)

---

## Invoice Workflow

### Sales Invoice (Customer Billing)

**Trigger:** A Trip reaches `Completed` status.

If `auto_create_sales_invoice = Yes` in Transport Setting, a **draft Sales Invoice** is created automatically and appears in your queue under **Accounts → Sales Invoice** (filter: Status = Draft).

If auto-create is off, generate manually:

1. Open the Transport Order (Status = `In Progress` or `Completed`)
2. Click **Create Sales Invoice**
3. Review line items:
    - Freight Linehaul
    - Fuel Surcharge (if applicable)
    - Detention (if applicable)
4. Verify VAT (15%) is applied
5. Click **Submit**

The invoice posts the accounting entry and updates the customer's outstanding balance.

### Purchase Invoice (Subcontractor Cost)

For outsourced trips:

1. Open the completed Trip (ownership = `Outsourced`)
2. Click **Create Purchase Invoice**
3. The invoice pre-fills with `Subcontractor Freight Cost` at the `subcon_line_total` amount
4. Verify the supplier (auto-filled from trailer's subcontractor field)
5. Submit

### GL Journal Entry (Owned-Fleet Costs)

Posted automatically when a `Completed` trip is submitted. Each row in Trip Costs becomes a debit line against the configured expense accounts.

Review posted entries: **Accounts → Journal Entry** → filter by `Posting Date` and `voucher_type = Journal Entry`.

---

## Rate Card Management

### Creating a Customer Rate Card

1. Go to **Transport Pricing → Customer Rate Card → New**
2. Set:
    - **Customer** — leave blank for a default card that applies to all customers
    - **Valid From / Valid To** — the period this card covers
    - **Status** → `Active`
3. Add lines in the **Lines** table, one per route + trailer type combination:

| Field | Guidance |
|---|---|
| Route | The Transport Route (e.g. `Riyadh IC → Jeddah Port`) |
| Trailer Type | Must match what customers will order |
| Base Rate | Linehaul for 1 pickup + 1 drop |
| Extra Stop Charge | Per stop beyond the standard 2 |
| Detention Free Hours | Grace period (e.g. 4 hours) |
| Detention Rate/Hr | Hourly charge after free hours |
| Fuel Surcharge % | Applied to linehaul; update monthly with fuel index |
| Min Charge | Floor amount; protects against very short trips |

4. Save and set Status to **Active**

!!! warning "Avoid Overlapping Dates"
    If two rate cards cover the same customer + route + trailer type for overlapping date ranges, the system picks the first one returned by query. Use `valid_to` dates carefully to avoid ambiguity.

### Updating Fuel Surcharge

Fuel surcharge is a percentage on each rate line. To update monthly:

1. Open the active rate card
2. Edit the `fuel_surcharge_pct` on each line
3. Save

Existing submitted orders are **not affected** — the surcharge is locked at order submission. New orders from the save date forward will use the updated rate.

### Subcontractor Rate Cards

Same structure as Customer Rate Cards, but keyed by **Supplier** and using cost field names (`base_cost`, `extra_stop_cost`, etc.). Maintain these alongside customer rates to track per-trip margin.

---

## Reports

Access all reports from **Transport Operations → Reports** or the Transport Operations Workspace.

### Trip Summary

Shows each trip with: order reference, customer, route, trailer, driver, planned/actual dates, selling amount, buying cost, profit, margin %.

**Use it for:** Monthly billing reconciliation, driver performance, route audit.

### Trip Profitability

Groups trips by route, customer, or trailer type. Shows total revenue, total cost, gross margin.

**Use it for:** Identifying unprofitable routes, renegotiating rate cards, fleet mix decisions.

### Cost Analysis

Breaks down owned-fleet costs (fuel, driver, tolls) by trip, trailer, or date range.

**Use it for:** Monitoring cost per km, budgeting fuel spend, toll reimbursement claims.

### Un-invoiced Completed Trips

Lists trips that have reached `Completed` status but have no linked Sales Invoice.

**Run daily.** Revenue leakage shows here.

### Outstanding Subcontractor Costs

Lists outsourced trips with no linked Purchase Invoice.

**Run weekly.** Avoid aging payables from vendor delays.

### Fleet Utilization

Shows each owned trailer: number of trips, total km, idle days in the period.

**Use it for:** Asset productivity, lease/own decisions, maintenance scheduling.

---

## Month-End Close Checklist

- [ ] Run **Un-invoiced Completed Trips** — invoice all completed trips
- [ ] Run **Outstanding Subcontractor Costs** — submit all pending Purchase Invoices
- [ ] Verify GL: Freight Revenue account balance = sum of submitted Sales Invoice amounts for the period
- [ ] Review **Trip Profitability** report — flag margin outliers for management review
- [ ] Update fuel surcharge % on rate cards for next month
- [ ] Archive expired rate cards (set Status → `Expired`)

---

## VAT Compliance (KSA)

- All customer Sales Invoices carry **15% VAT** via the KSA Sales Taxes template
- ZATCA Phase-2 e-invoicing is handled by the ERPNext KSA compliance add-on, not this app
- Purchase Invoices for VAT-registered subcontractors also apply input VAT — verify supplier VAT registration status before submitting
- If a customer is VAT-exempt, apply an exemption tax template at the order level before submission
