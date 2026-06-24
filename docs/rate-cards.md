# Rate Cards

**Audience:** Transport Finance and administrators.

Rate cards are the pricing engine's source of truth. Every Transport Order line price and every Trip buying cost originates from a rate card line. Understanding the resolution logic saves hours of debugging.

---

## Rate Card Types

| Type | Doctype | Keys | Used for |
|---|---|---|---|
| Customer | `Customer Rate Card` | customer + route + trailer type | Selling price on Transport Order |
| Subcontractor | `Subcontractor Rate Card` | subcontractor + route + trailer type | Buying cost on Trip |

Both types share the same line structure and the same pricing formula.

---

## Resolution Precedence (Customer Rate Cards)

When an Order Item is saved, the engine searches for a rate in this order:

1. **Customer-specific card** valid on the order date, with a line matching route + trailer type
2. **Default card** (Customer = blank) valid on the order date, with a matching line

If neither exists, the order item cannot be priced and the save fails with a clear error.

### Example

Order date: 2026-03-15, Customer: Arabian Logistics, Route: Riyadh IC → Jeddah Port, Type: Flatbed 40T

```
Cards checked:
  1. CRC-2026-00001 (customer=Arabian Logistics, valid 2026-01-01 to 2026-12-31)
     → Has line for Riyadh IC → Jeddah Port / Flatbed 40T? YES → USE THIS
  2. (not checked — found in step 1)
```

If step 1 had no matching line:
```
  2. CRC-2026-00002 (customer=blank, valid 2026-01-01 to 2026-12-31)
     → Has line for Riyadh IC → Jeddah Port / Flatbed 40T? YES → USE THIS
```

---

## Pricing Formula (Both Card Types)

```
extra_stops       = max(0, total_stops − 2)
linehaul          = base_rate + (extra_stops × extra_stop_charge)
detention_amount  = max(0, detention_hours − detention_free_hours)
                    × detention_rate_per_hour
fuel_surcharge    = linehaul × fuel_surcharge_pct / 100
line_total        = max(min_charge, linehaul + detention_amount + fuel_surcharge)
                    × qty_trailers
```

See [Pricing Engine](pricing-engine.md) for full Python implementation.

---

## Setting Line Values

### Base Rate

The linehaul charge for a standard 2-stop move (1 pickup + 1 drop). This is your core rate negotiated with the customer.

### Extra Stop Charge

Charged **per additional stop** beyond the standard 2. If a customer regularly has 3-stop deliveries, set this to cover the incremental time and distance.

Example:
- Base Rate: SAR 2,000 (covers 1 pickup + 1 drop)
- Extra Stop Charge: SAR 300
- Order with 3 stops: SAR 2,000 + (1 × SAR 300) = SAR 2,300

### Detention Free Hours and Rate

Detention is charged when a driver waits at a site beyond the free grace period.

- `detention_free_hours = 4` means the first 4 hours are free
- `detention_rate_per_hour = SAR 150` means hour 5+ costs SAR 150/hr
- Dispatcher enters `detention_hours` on the trip/order when the driver reports waiting

### Fuel Surcharge %

Applied as a percentage of the linehaul amount. Update monthly based on your fuel cost index.

**Best practice:** Create a new rate card version (or simply update the pct on the active card) at the start of each month. Submitted orders retain their locked rate.

### Min Charge

Floor amount per trailer. Prevents below-cost pricing on very short or light trips.

---

## Managing Rate Card Versions

When rates change (new contract, fuel index update):

**Option A — Update in place** (if only fuel surcharge changes):
- Open the active card, edit `fuel_surcharge_pct` on each line, save.
- Submitted orders are not affected.

**Option B — New card version** (for full renegotiation):
1. Set the old card `valid_to` to the last day of the old period
2. Create a new card with `valid_from` = first day of new period
3. Set new card status to `Active`
4. Set old card status to `Expired`

---

## Troubleshooting Rate Cards

| Problem | Likely cause | Fix |
|---|---|---|
| "No active Customer Rate Card found" | Card expired, wrong date range, or missing line | Check valid_from/to; add missing line |
| Wrong price auto-filled | Customer-specific card overrides default unexpectedly | Review which card is active for that customer |
| Margin shows negative | Customer rate < subcontractor rate | Check both rate cards for the route; renegotiate |
| Rate doesn't update after card change | Old rate locked on submitted order | Cancel and amend the order |
| Fuel surcharge = 0 despite pct set | `apply_fuel_surcharge_on` misconfigured | Check Transport Setting |
