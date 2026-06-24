# Roles & Permissions

**Audience:** System administrators.

Enigma TMS defines 4 custom roles installed via `fixtures/role.json`. Assign them in ERPNext under Settings → User.

---

## Role Capabilities

| Capability | Order Desk | Dispatcher | Transport Finance | Transport Manager |
|---|---|---|---|---|
| Create Transport Order | ✓ | — | — | ✓ |
| Submit Transport Order | ✓ | — | — | ✓ |
| Cancel Transport Order | — | — | — | ✓ |
| Read Transport Order | ✓ | ✓ | ✓ | ✓ |
| Create Trip | — | ✓ | — | ✓ |
| Submit Trip | — | ✓ | — | ✓ |
| Update Trip Costs | — | ✓ | ✓ | ✓ |
| Cancel Trip | — | — | — | ✓ |
| Read Trip | ✓ | ✓ | ✓ | ✓ |
| Create/Edit Rate Cards | — | — | ✓ | ✓ |
| Create Sales Invoice | — | — | ✓ | ✓ |
| Create Purchase Invoice | — | — | ✓ | ✓ |
| View Reports | Read-only | Read-only | ✓ | ✓ |
| Transport Setting | — | — | — | ✓ |
| Manage Users | — | — | — | ✓ (via System Manager) |

---

## Permission Table (Frappe DocPerm)

| Doctype | Role | Read | Write | Submit | Cancel | Delete |
|---|---|---|---|---|---|---|
| Transport Order | Order Desk | ✓ | ✓ | ✓ | — | — |
| Transport Order | Dispatcher | ✓ | — | — | — | — |
| Transport Order | Transport Finance | ✓ | — | — | — | — |
| Transport Order | Transport Manager | ✓ | ✓ | ✓ | ✓ | ✓ |
| Trip | Order Desk | ✓ | — | — | — | — |
| Trip | Dispatcher | ✓ | ✓ | ✓ | — | — |
| Trip | Transport Finance | ✓ | ✓ | — | — | — |
| Trip | Transport Manager | ✓ | ✓ | ✓ | ✓ | ✓ |
| Customer Rate Card | Transport Finance | ✓ | ✓ | — | — | — |
| Customer Rate Card | Transport Manager | ✓ | ✓ | — | — | ✓ |
| Subcontractor Rate Card | Transport Finance | ✓ | ✓ | — | — | — |
| Trailer / Tractor | Dispatcher | ✓ | ✓ | — | — | — |
| Trailer / Tractor | Transport Manager | ✓ | ✓ | — | — | ✓ |
| Transport Setting | Transport Manager | ✓ | ✓ | — | — | — |

---

## Adding a Custom Role

If you need an additional role (e.g. "Driver Supervisor"):

1. ERPNext → Settings → Role → New
2. Name the role
3. Go to each relevant Doctype → Permissions → Add the role with required levels
4. Or add via `fixtures/custom_docperm.json` if you want the permissions to persist across re-installs

---

## Field-Level Security

Some fields are conditionally read-only based on document status, enforced server-side via `allow_on_submit`:

- `Transport Order Item` pricing fields — read-only after submit
- `Trip.transport_order` — read-only after submit
- `Trip.trailer` / `Trip.driver` — can only change via Cancel+Amend

These are not role-based — they apply to all users.
