# Transport Manager Guide

**Role:** Transport Manager
**Can:** Full access to all doctypes, reports, settings, and user management
**Use this guide for:** Dashboard interpretation, KPI monitoring, strategic reporting, configuration oversight

---

## Dashboard Overview

The **Transport Operations Workspace** (your home screen) shows 11 KPI cards and 9 charts.

### KPI Cards

| Card | What it measures | Healthy range |
|---|---|---|
| **Active Trips** | Trips currently in progress (status not Completed/Cancelled) | Depends on fleet size |
| **Trailers Available** | Owned trailers with status = Available | > 20% of fleet |
| **On-Time Delivery %** | Trips where actual_end ≤ planned_end / total completed | > 90% |
| **Revenue This Week** | Sum of submitted Sales Invoices this calendar week | Compare to target |
| **Trip Margin %** | Avg (selling − buying) / selling across completed trips this month | > 25% |
| **Un-invoiced Trips** | Completed trips without a Sales Invoice | Should be 0 |
| **Pending POD** | Trips in Delivered status (POD uploaded, awaiting final Completed) | Should be < 5 |
| **Trailers On Trip** | Owned trailers currently assigned | Fleet utilization indicator |
| **Overdue Trips** | Trips where planned_end < now and status ≠ Completed | Should be 0 |
| **Open Transport Orders** | Orders in Confirmed status not yet converted to a Trip | Dispatch backlog |
| **Cost Per Trip (Avg)** | Average buying_total across owned-fleet trips this month | Track trend |

### Dashboard Charts

| Chart | Type | Key insight |
|---|---|---|
| Revenue by Week | Bar/Line time-series | Revenue trend and seasonality |
| Trip Volume by Status | Donut | Where your trips are right now |
| Top 10 Routes by Revenue | Bar | Most valuable lanes |
| Owned vs Outsourced Trips | Pie | Fleet mix; outsourcing rate |
| Trailer Utilization | Bar (per trailer) | Idle assets |
| Trip Margin Distribution | Histogram | Margin spread; outlier detection |
| Daily Active Trips | Time-series | Operational peak/trough |
| Cost Breakdown (Fuel/Driver/Toll) | Stacked bar | Expense composition |
| Revenue vs Cost by Month | Grouped bar | Monthly P&L summary |

---

## Strategic Reports

### Route Profitability

Go to **Reports → Route Profitability**. Group by Route, sort by Margin % descending.

- **Highest margin routes** — consider increasing capacity allocation
- **Lowest/negative margin routes** — rate card may be outdated; investigate cost drivers
- Filter by date range to isolate fuel-spike periods

### Fleet Utilization

Go to **Reports → Fleet Utilization**. Shows each owned trailer: trips, km, idle days.

- Trailers with > 30 idle days in a month: investigate — maintenance backlog or booking gap?
- High km trailers: check if maintenance is keeping pace

### Customer Profitability

Cross-reference Trip Profitability report filtered by Customer with Accounts Receivable aging.

- High-volume, low-margin customers: renegotiate rate cards
- High-margin, low-volume: growth opportunity; prioritize service level

---

## Configuration Responsibilities

As Transport Manager you own:

| Setting | Where | Review frequency |
|---|---|---|
| Transport Setting (global toggles) | Transport Setup → Transport Setting | Quarterly |
| Rate card validity periods | Transport Pricing → Customer/Sub Rate Card | Monthly (fuel surcharge) |
| Driver license expiry | Transport Setup → Driver | Ongoing |
| Trailer maintenance status | Transport Setup → Trailer | After each maintenance event |
| User roles | ERPNext HR / Users | When staff change |

---

## Escalation Paths

| Situation | Action |
|---|---|
| Customer disputes invoice amount | Compare Transport Order pricing to Rate Card; check if amendment was missed |
| Trip margin < 10% | Review Trip Costs — fuel or driver cost spike? Subcon rate mismatch? |
| Trailer availability < 15% | Check Trailers On Trip chart — are trips completing on time? |
| Un-invoiced Trips > 10 | Escalate to Finance; revenue recognition gap |
| Driver license expired | Block driver from new trip assignments; coordinate renewal |
