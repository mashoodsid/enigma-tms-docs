# Finance Integration

**Audience:** Finance staff and Frappe developers. Assumes ERPNext accounting basics.

Enigma TMS generates three types of financial documents in ERPNext, all owned by standard doctypes. The TMS app creates them; the accounting team manages them.

---

## Sales Invoice (Customer)

**When created:** Automatically (if `auto_create_sales_invoice = Yes` in Transport Setting) when a Trip reaches `Completed`, or manually via the **Create Sales Invoice** button on the Transport Order.

**Line items** (non-stock service Items):

| Item | Populated from |
|---|---|
| Freight Linehaul | `Transport Order Item.line_total` base + extra stops |
| Fuel Surcharge | `Transport Order Item.fuel_surcharge_amount` |
| Detention | `Transport Order Item.detention_amount` (if > 0) |

**Tax:** KSA VAT 15% via the `default_vat_template` in Transport Setting.

**Accounting entry (auto by ERPNext):**

```
Dr  Accounts Receivable (Customer)   2,875 SAR
    Cr  Freight Revenue Account       2,500 SAR
    Cr  VAT Payable (Output Tax)        375 SAR
```

---

## Purchase Invoice (Subcontractor)

**When created:** Only for Trips where `ownership = Outsourced`. Available via **Create Purchase Invoice** on the Trip form after `Completed` status.

**Line items:** `Subcontractor Freight Cost` at the `subcon_line_total` resolved from the Subcontractor Rate Card.

**Accounting entry:**

```
Dr  Freight Cost — Outsourced        1,200 SAR
    Cr  Accounts Payable (Supplier)  1,200 SAR
```

---

## GL Journal Entry (Owned-Fleet Costs)

For **owned-fleet trips**, direct costs (fuel, driver pay, tolls) are posted as a Journal Entry when the Trip is submitted with a `Completed` status.

Each row in `Trip Costs` becomes a debit line:

```
Dr  Fuel Expense Account              450 SAR   (row: Fuel)
Dr  Driver Pay Expense Account        300 SAR   (row: Driver Pay)
Dr  Toll Expense Account               75 SAR   (row: Toll)
    Cr  Intercompany Payable / Cash   825 SAR
```

The credit account is configurable in Transport Setting. If your workflow uses petty cash disbursements, point the credit to the petty cash account.

---

## Trip Profit & Loss

The Trip form always shows:

| Field | Calculation |
|---|---|
| `selling_amount` | From parent Transport Order `selling_total` |
| `buying_total` | Sum of `Trip Costs.amount` (owned) OR `subcon_line_total` (outsourced) |
| `trip_profit` | `selling_amount − buying_total` |
| `trip_margin_pct` | `trip_profit / selling_amount × 100` |

This is a real-time view — it does not depend on GL posting being complete.

---

## VAT & ZATCA Notes

- All Sales Invoices use **15% KSA VAT** via the configured Sales Taxes and Charges Template
- The app does **not** handle ZATCA Phase-2 e-invoicing directly — use the [KSA ERPNext compliance app](https://github.com/frappe/erpnext/tree/version-15/erpnext/regional/saudi_arabia) alongside this app
- Purchase Invoices for subcontractors follow the same VAT template unless the supplier is VAT-exempt

---

## Month-End Close Checklist

1. **Un-invoiced Trips report** — run from Reports menu; identify completed trips without a Sales Invoice
2. **Outstanding Subcontractor Costs** — run report; match to Purchase Invoices submitted
3. **Reconcile GL** — verify Freight Revenue account balance matches sum of submitted Sales Invoice amounts
4. **Trip Profit report** — review margins by route; investigate outliers before closing period
