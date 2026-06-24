# Troubleshooting & FAQs

---

## Common Errors

### "No active Customer Rate Card found"

**Symptom:** Error when saving a Transport Order item.

**Causes:**
1. No rate card exists for this route + trailer type combination
2. Rate card exists but `valid_to` is in the past
3. Rate card exists but `status ≠ Active`
4. Order date is outside the card's `valid_from` / `valid_to` range

**Fix:**
- Go to **Transport Pricing → Customer Rate Card**
- Search for the customer (or default card with blank customer)
- Check status = `Active` and dates cover the order date
- If no matching line exists, add one

---

### "Trailer already assigned to another trip"

**Symptom:** Error at Trip submit.

**Cause:** Another active trip (not Completed or Cancelled) has the same trailer with overlapping `planned_start` / `planned_end`.

**Fix:**
- Check the conflicting trip: **Transport Operations → Trip → filter by trailer**
- Either: complete or cancel the conflicting trip, or choose a different trailer
- If the conflict is a data error, cancel the incorrect trip and re-submit

---

### "Customer credit limit exceeded"

**Symptom:** Warning or block at Transport Order submit.

**Fix:**
- Go to ERPNext → Accounts → Customer → [Customer] → Credit Limit
- Either increase the credit limit, or get Finance to clear outstanding invoices
- Transport Manager can override if `credit_check_on_order = Warn` (not blocked)

---

### Trip profit shows SAR 0

**Symptom:** `trip_profit = 0` despite valid selling and buying amounts.

**Cause:** Usually the buying cost is not entered (owned trip with empty Trip Costs table), or the subcontractor rate card is not found (outsourced trip).

**Fix (Owned):** Enter at least one row in Trip Costs. `buying_total` = sum of cost rows.

**Fix (Outsourced):** Verify the Subcontractor Rate Card has an active line for this route + trailer type. Check `subcon_line_total` field on the Trip form — it should be populated after save.

---

### Sales Invoice not created automatically

**Symptom:** Trip reaches `Completed` but no draft Sales Invoice appears.

**Cause:** `auto_create_sales_invoice = No` in Transport Setting, or the trigger hook errored.

**Fix:**
- Check Transport Setting → `auto_create_sales_invoice`
- Check **Error Log** in ERPNext for any hook error on the Trip submit
- Create the invoice manually via **Create Sales Invoice** button on the Transport Order

---

### Trailer status stuck at "On Trip"

**Symptom:** A trailer shows `On Trip` but has no active trip.

**Cause:** The Trip was cancelled without the `on_cancel` hook running (e.g. a database-level delete, or a hook error).

**Fix:**

```python
# From bench console
bench --site your-site.com console

import frappe
trailer = frappe.get_doc("Trailer", "TRL-00001")
trailer.status = "Available"
trailer.save()
frappe.db.commit()
```

---

### "Stops table is empty" on submit

**Symptom:** Cannot submit a Transport Order.

**Cause:** No stops added to the Stops child table.

**Fix:** Add at least one Pickup stop and one Drop stop in the Stops section of the Transport Order form.

---

### Dashboard charts show no data

**Symptom:** Charts display "No data" after going live.

**Cause:** Charts filter on `docstatus = 1` (submitted documents). Draft documents don't appear.

**Fix:** Submit at least one Transport Order and one Trip. Charts update on the next dashboard refresh (default 5 minutes, or press F5).

---

## FAQ

**Q: Can one Transport Order have multiple Trips?**

Yes. One order can spawn multiple Trips (e.g. different trailers for different lanes, or a repeat run). The order status moves to `Completed` only when all linked trips are `Completed`.

**Q: Can I change the trailer on a submitted Trip?**

No — submitted documents are locked. Cancel and amend the Trip to change the trailer or driver.

**Q: What happens if I delete a Rate Card that's in use?**

Submitted orders retain their locked pricing (the rate card values are copied to the order item at submission). Only new orders are affected by rate card changes. However, deleting an active card is bad practice — set it to `Expired` instead.

**Q: How do I handle a trip that was partially delivered?**

Mark completed stops as `Completed` with PODs. Leave incomplete stops as `Pending`. Do not submit the Trip as `Completed` — it will be stuck in `Delivered` or `In Transit`. Contact the Transport Manager to decide whether to re-attempt delivery or cancel the remaining stops.

**Q: Can I import Transport Orders in bulk?**

Use ERPNext's standard **Data Import** tool (Tools → Data Import). The Transport Order and Transport Order Item doctypes support import. For multi-stop orders, use a separate import for the Transport Stop child table referencing the same parent order name. A custom importer with Order Ref grouping is planned for V2.

**Q: Where do I find the ZATCA e-invoice integration?**

ZATCA Phase-2 e-invoicing is not part of this app. Use the [ERPNext KSA localisation](https://github.com/frappe/erpnext/tree/version-15/erpnext/regional/saudi_arabia) or a dedicated compliance add-on alongside Enigma TMS.

**Q: My bench has two sites (tms + fms). Will installing transport_management affect the fms site?**

No. Frappe apps are installed per-site. Running `bench --site tms.yourdomain.com install-app transport_management` only affects the `tms` site. The `fms` site is unaffected.

**Q: How do I reset demo data for testing?**

```bash
bench --site tms.yourdomain.com execute transport_management.setup.demo_data.clear
bench --site tms.yourdomain.com execute transport_management.setup.demo_data.seed
```

!!! warning
    `clear` drops all TMS transaction data (orders, trips, invoices). Do not run on production.
