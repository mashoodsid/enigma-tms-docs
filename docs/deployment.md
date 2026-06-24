# Deployment Guide

**Audience:** DevOps engineers and system administrators deploying Enigma TMS to production.

---

## Production Environment (Current)

| Item | Value |
|---|---|
| Server | Contabo VPS (4 vCPU / 8 GB RAM) |
| OS | Ubuntu 22.04 LTS |
| Frappe | v15.56.1 |
| ERPNext | v15.x |
| Python | 3.10 |
| MariaDB | 10.6 |
| URL | tms.enigmaerp.com |
| Bench path | `/home/frappe/frappe-bench` |

---

## Fresh Server Setup

### 1. Prerequisites

```bash
# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install Frappe Easy Install dependencies
sudo apt-get install -y git curl python3-pip

# Install Frappe Easy Install (sets up bench, MariaDB, Redis, Nginx, Supervisor)
curl https://raw.githubusercontent.com/frappe/bench/develop/install.py | python3 -
```

Or use the official [frappe/bench easy install script](https://frappeframework.com/docs/user/en/installation) for Frappe v15.

### 2. Create a New Bench

```bash
bench init frappe-bench --frappe-branch version-15
cd frappe-bench
bench get-app erpnext --branch version-15
bench get-app https://github.com/your-org/transport_management
```

### 3. Create the Site

```bash
bench new-site tms.yourdomain.com \
  --db-name tms_db \
  --admin-password YOUR_ADMIN_PASSWORD
bench --site tms.yourdomain.com install-app erpnext
bench --site tms.yourdomain.com install-app transport_management
bench --site tms.yourdomain.com migrate
```

### 4. Set Up Production

```bash
sudo bench setup production frappe  # sets up Nginx + Supervisor
bench setup nginx
sudo nginx -t && sudo systemctl reload nginx
```

### 5. SSL Certificate

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d tms.yourdomain.com
```

Let's Encrypt auto-renews via cron. Verify:

```bash
sudo certbot renew --dry-run
```

---

## Production Checklist

### Infrastructure
- [ ] Server hardened: SSH key-only, UFW firewall (22, 80, 443 only)
- [ ] Swap configured (at least equal to RAM)
- [ ] Monitoring agent installed (e.g. Netdata, Datadog)

### Web / DNS
- [ ] DNS A record → server IP
- [ ] SSL certificate issued and auto-renewing
- [ ] Nginx reverse proxy config verified (`bench setup nginx`)
- [ ] HTTPS redirect enabled

### Application
- [ ] All apps installed: erpnext, transport_management
- [ ] `bench migrate` completed with no errors
- [ ] `bench build` completed (assets compiled)
- [ ] Transport Setting configured (VAT template, GL accounts)
- [ ] At least one user with Transport Manager role

### Backup
- [ ] Automated backups configured (see Backup section)
- [ ] Off-server backup destination configured (S3, SFTP)
- [ ] Test restore verified

### Security
- [ ] Admin password changed from default
- [ ] `site_config.json` has `allow_cors = 0` unless API access needed
- [ ] ERPNext `System Settings → Password Policy` configured

---

## Backup & Recovery

### Automated Backup (Bench Built-in)

Frappe takes daily backups automatically. Default location:

```
/home/frappe/frappe-bench/sites/tms.yourdomain.com/private/backups/
```

Configure off-server copy (add to `site_config.json`):

```json
{
  "backup_path": "/path/to/backups",
  "s3_backup_path": "s3://your-bucket/tms-backups/"
}
```

### Manual Backup

```bash
bench --site tms.yourdomain.com backup --with-files
```

Creates:
- `*-database.sql.gz` — MariaDB dump
- `*-files.tar` — uploaded files
- `*-private-files.tar` — private attachments

### Restore

```bash
# Stop bench workers first
bench stop  # or sudo supervisorctl stop all

# Restore DB
bench --site tms.yourdomain.com --force restore \
  /path/to/20260615_120000-tms_db-database.sql.gz

# Restore files
tar -xf /path/to/20260615_120000-tms_db-files.tar \
  -C /home/frappe/frappe-bench/sites/tms.yourdomain.com/

bench --site tms.yourdomain.com migrate
bench start  # or sudo supervisorctl start all
```

---

## Upgrading

```bash
cd /home/frappe/frappe-bench

# Backup first
bench --site tms.yourdomain.com backup --with-files

# Pull updates
bench update --apps transport_management

# Migrate
bench --site tms.yourdomain.com migrate

# Restart
sudo supervisorctl restart all
```

For Frappe/ERPNext major version upgrades, follow the official [Frappe upgrade guide](https://frappeframework.com/docs/user/en/versions).

---

## Monitoring & Logs

### Key Log Files

```bash
# Frappe web server
tail -f /home/frappe/frappe-bench/logs/web.error.log

# Background workers
tail -f /home/frappe/frappe-bench/logs/worker.error.log

# Scheduler
tail -f /home/frappe/frappe-bench/logs/schedule.error.log

# Nginx
tail -f /var/log/nginx/error.log
```

### Common Error Patterns

| Log message | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` | Gunicorn/Redis down | `sudo supervisorctl restart frappe*` |
| `OperationalError: 1040 Too many connections` | DB pool exhausted | Reduce `worker_per_core` in site_config or scale DB |
| `ERPNextValidationError: rate card not found` | Missing rate card line | Add the line to the active rate card |
| `frappe.DoesNotExistError: Transport Order` | Stale link or wrong site | Verify you're on the correct site |
| `RateLimitExceeded` | API being hammered | Add rate-limiting in Nginx or implement API key access |

### Performance Tuning

```json
// site_config.json
{
  "db_pool_size": 10,
  "db_max_overflow": 5,
  "worker_pool_size": 4,
  "background_workers": 2,
  "enable_db_cache": true
}
```

---

## Multi-Site (fms + tms on Same Bench)

The current Contabo bench runs both `tms.enigmaerp.com` and `fms.enigmaerp.com`.

Key points:
- Each site has its own database and `site_config.json`
- Apps are shared at bench level — installing `transport_management` on one site does not affect the other
- Nginx uses `server_name` to route requests to the correct site
- Background workers are shared — heavy jobs on one site can affect the other

To add a new site to an existing bench:

```bash
bench new-site newsite.yourdomain.com
bench --site newsite.yourdomain.com install-app erpnext
bench --site newsite.yourdomain.com install-app transport_management
bench setup nginx  # regenerate nginx config to add new site
sudo nginx -t && sudo systemctl reload nginx
```

---

## Roadmap Callouts (V2)

| Feature | Status | Notes |
|---|---|---|
| ZATCA Phase-2 e-invoicing | Planned | Requires KSA ERPNext compliance app integration |
| Live fleet map | Planned (P2) | Leaflet/MapLibre custom page; needs GPS/Traccar source |
| Driver PWA | Planned (P2) | Mobile stop updates and POD capture |
| Auto-dispatch optimizer | Future | VROOM-based route optimization |
| Cross-border customs clearance | Future | Activate `is_cross_border` flag on routes |
| Customer self-service portal | Future | Frappe web portal with order tracking |

APIs marked with `# P2` in `api-reference.md` are stubs for upcoming features and should not be relied upon in the current version.
