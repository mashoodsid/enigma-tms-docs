# Installation

**Audience:** Frappe server administrators. Prerequisites: a Frappe v15 bench already running.

---

## System Requirements

| Component | Minimum | Recommended |
|---|---|---|
| Frappe | v15.56.1 | Latest v15 |
| Python | 3.10 | 3.11 |
| MariaDB | 10.6 | 10.11 |
| Node.js | 18 | 20 LTS |
| Redis | 6 | 7 |
| RAM | 4 GB | 8 GB |
| Disk | 20 GB | 50 GB SSD |

ERPNext must be installed on the same site — `transport_management` depends on `erpnext` for Customer, Driver, Sales Invoice, Purchase Invoice, and GL Journal Entry doctypes.

---

## Installation Steps

### 1. Get the App

```bash
cd /path/to/your/bench
bench get-app https://github.com/your-org/transport_management
```

### 2. Install on Your Site

```bash
bench --site your-site.com install-app transport_management
```

### 3. Run Migrations

```bash
bench --site your-site.com migrate
```

This creates all 16 doctypes, naming series, and fixtures (roles, custom fields on Driver).

### 4. Build Assets

```bash
bench build --app transport_management
bench restart
```

### 5. Post-Install Checklist

After installation, complete these steps before going live:

- [ ] **Company Setup** — ensure your company has a Chart of Accounts (CoA) and fiscal year configured in ERPNext
- [ ] **Transport Setting** — open `Transport Setting` (Single doctype) and configure:
    - Default VAT template (KSA 15%)
    - Default GL accounts for freight revenue and trip costs
    - Auto-create invoice on trip completion: Yes/No
- [ ] **Naming Series** — verify naming series in `Transport Order`, `Trip`, `Transport Location`, `Transport Route` match your conventions
- [ ] **Roles** — assign users to `Order Desk`, `Dispatcher`, `Transport Finance`, or `Transport Manager`
- [ ] **Seed Master Data** — create at least one Location, Route, Trailer Type, Trailer, Driver, and Rate Card before creating orders

---

## Transport Setting (Single Doctype)

This is the global configuration for the app. Navigate to **Transport Setup → Transport Setting**.

| Field | Description |
|---|---|
| `apply_fuel_surcharge_on` | Whether fuel surcharge % applies to `Linehaul` only or full subtotal |
| `default_vat_template` | Sales Taxes template for KSA 15% VAT |
| `default_revenue_account` | GL account for freight revenue lines on Sales Invoices |
| `owned_fleet_cost_account` | GL account for owned-trip expenses (fuel, driver, tolls) |
| `auto_create_sales_invoice` | Create draft Sales Invoice automatically when a Trip reaches `Completed` |
| `credit_check_on_order` | Block/warn when customer exceeds credit limit at Order submission |

---

## Upgrading

```bash
bench update --apps transport_management
bench --site your-site.com migrate
bench restart
```

Always back up before upgrading:

```bash
bench --site your-site.com backup --with-files
```

---

## Uninstalling

```bash
bench --site your-site.com uninstall-app transport_management
```

!!! warning
    Uninstalling removes all doctypes and data for `transport_management`. Back up first. ERPNext Sales/Purchase Invoices created by the app remain — they are standard ERPNext documents.
