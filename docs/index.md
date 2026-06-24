---
hide:
  - navigation
  - toc
---

# Enigma TMS

**Transport Management System for Saudi Arabia trailer fleet operations**

Enigma TMS is a standalone Frappe v15 application that manages the complete transport lifecycle for domestic road-transport businesses: **Order → Dispatch → Delivery → Invoicing**. Built for owned and outsourced trailer fleets, it delivers hybrid pricing, real-time dispatch visibility, and full ERPNext finance integration out of the box.

---

## Key Capabilities

<div class="grid cards" markdown>

-   :material-clipboard-text-clock: **Transport Orders**

    ---

    Multi-lane, multi-stop orders priced automatically from customer rate cards. No RFQ stage — work enters the system ready to dispatch.

-   :material-truck-delivery: **Trip Execution**

    ---

    Assign owned tractors/trailers or outsource to subcontractors. Track per-trip costs (fuel, driver pay, tolls) and compute margins in real time.

-   :material-file-document-check: **Proof of Delivery**

    ---

    Per-stop POD collection (photo, signature, QR scan) gates trip completion and invoice generation — no POD, no billing.

-   :material-receipt: **Finance Integration**

    ---

    Auto-generate Sales Invoices (15% KSA VAT), Purchase Invoices for subcontractors, and GL Journal Entries for owned-fleet costs — all inside ERPNext.

-   :material-chart-bar: **Dashboard & Reports**

    ---

    11 KPI cards, 9 charts, a Kanban dispatch board, and 6 business reports covering revenue, cost, utilization, and route profitability.

-   :material-cog: **Configurable Workflows**

    ---

    Trip status flows, rate card scoping, and POD requirements are configuration, not code — operations teams can adapt flows without a developer.

</div>

---

## Business Flow

```
Transport Order  ──►  Trip (Dispatch)  ──►  POD  ──►  Invoicing
  confirmed job         assign trailer       proof       Sales Invoice (customer)
  priced from           driver + costs       of          Purchase Invoice (sub)
  rate card             owned/outsourced     delivery  ► Trip profit = sell − buy
```

---

## Quick Navigation

| I want to… | Go to |
|---|---|
| Walk through end-to-end in 15 minutes | [Quick Start](getting-started.md) |
| Create a Transport Order | [Order Desk Guide](user-guides/order-desk.md) |
| Assign trips and manage dispatch | [Dispatcher Guide](user-guides/dispatcher.md) |
| Generate invoices and run reports | [Finance Guide](user-guides/finance.md) |
| Set up rate cards, routes, and assets | [Configuration Guide](configuration.md) |
| Troubleshoot a common issue | [Troubleshooting](troubleshooting.md) |

---

## System at a Glance

| Attribute | Value |
|---|---|
| Modules | Transport Setup · Transport Pricing · Transport Operations |
| Roles | Order Desk · Dispatcher · Transport Finance · Transport Manager |
| VAT | KSA 15% (ZATCA-ready) |
| Currency | SAR |
| Languages | English (Arabic localisation planned) |
