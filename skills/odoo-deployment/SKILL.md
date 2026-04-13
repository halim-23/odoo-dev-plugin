---
name: odoo-deployment
description: >
  Expert Odoo production deployment, server setup, and DevOps guidance for versions 17.0, 18.0, 19.0.
  Covers production server, odoo.conf, nginx, SSL, systemd, Docker, Odoo.sh, backup, restore,
  update strategy, multiple workers, pgbouncer, server migration, deployment pipeline, git submodule
  on Odoo.sh, staging branch, production branch, server crash, out of memory, high CPU, module update
  on production, or anything about running Odoo reliably in production. Also trigger for 'how do I
  deploy my module', 'set up a production Odoo server', 'configure SSL for Odoo', 'how to update
  production safely', 'Odoo.sh branch strategy', or 'Docker compose for Odoo'.
---

Expert Odoo deployment engineer. Target: 17/18/19 CE+EE.

**Always confirm** hosting type (Odoo.sh, self-hosted VPS, Docker, on-premise) and Odoo version before giving deployment advice.

---

## Deployment Options Overview

```
Odoo.sh          → Managed PaaS by Odoo — easiest, git-based
Self-hosted VPS  → Full control, most common for custom modules
Docker           → Reproducible, good for CI/CD pipelines
On-premise       → Customer's own servers
```

---

## 1. Odoo.sh — Branch Strategy

Odoo.sh gives you three branch types: **Production**, **Staging**, **Development**.

```
main (Production)
  └── staging (Staging)
        └── feature/your-feature (Development)

Workflow:
1. Develop in a dev branch (auto-deploys on each push)
2. Merge to staging → test with production data copy
3. Merge to main (Production) → live update
```

### Custom modules on Odoo.sh

```bash
# Option A: Submodule (recommended for private repos)
cd your-odoo-sh-repo
git submodule add https://github.com/yourorg/your_module.git
git add .gitmodules your_module/
git commit -m "[ADD] your_module: add as submodule"
git push

# Option B: Direct in repo
cp -r your_module/ ./
git add your_module/
git commit -m "[ADD] your_module: add custom module"
git push

# Python dependencies (requirements.txt in repo root)
echo "your-python-package==1.2.3" >> requirements.txt
git add requirements.txt
git commit -m "[IMP] requirements: add your-python-package"
git push
```

### Odoo.sh config file (`.odoorc` in repo root)

```ini
[options]
list_db = False
log_level = warn
```

### Odoo.sh hooks (`hooks.py` in module root)

```python
def pre_init_hook(env):
    """Runs before module install."""
    pass

def post_init_hook(env):
    """Runs after module install."""
    pass

def uninstall_hook(env):
    """Runs before module uninstall."""
    pass
```

```python
# __manifest__.py
{
    'pre_init_hook': 'pre_init_hook',
    'post_init_hook': 'post_init_hook',
    'uninstall_hook': 'uninstall_hook',
}
```

---

## 2. Self-Hosted VPS — Full Setup (Ubuntu 22.04)

```bash
# System dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-dev python3-venv \
    build-essential libxslt-dev libzip-dev libldap2-dev \
    libsasl2-dev libssl-dev git wget curl \
    postgresql postgresql-contrib \
    wkhtmltopdf nginx

# Create odoo user
sudo adduser --system --group --home /opt/odoo odoo

# PostgreSQL user for Odoo
sudo -u postgres createuser --superuser odoo

# Clone Odoo (example: 18.0)
sudo git clone --depth 1 --branch 18.0 \
    https://github.com/odoo/odoo.git /opt/odoo/odoo

# Python venv
sudo python3 -m venv /opt/odoo/venv
sudo /opt/odoo/venv/bin/pip install -r /opt/odoo/odoo/requirements.txt
sudo /opt/odoo/venv/bin/pip install psycopg2-binary

# Custom addons directory
sudo mkdir /opt/odoo/custom-addons
sudo chown odoo:odoo /opt/odoo/custom-addons
```

### `/etc/odoo/odoo.conf`

```ini
[options]
; ── Database ───────────────────────────────────────────────
admin_passwd = CHANGE_THIS_MASTER_PASSWORD
db_host = False
db_port = False
db_user = odoo
db_password = False

; ── Paths ──────────────────────────────────────────────────
addons_path = /opt/odoo/odoo/addons,/opt/odoo/custom-addons
data_dir = /opt/odoo/data
logfile = /var/log/odoo/odoo.log

; ── Workers ────────────────────────────────────────────────
workers = 4              ; (CPU × 2) + 1
max_cron_threads = 2

; ── Memory ─────────────────────────────────────────────────
limit_memory_soft = 2147483648    ; 2 GB
limit_memory_hard = 2684354560    ; 2.5 GB
limit_time_cpu = 60
limit_time_real = 120
limit_time_real_cron = 1800

; ── Security ───────────────────────────────────────────────
list_db = False          ; hide DB list on login page
proxy_mode = True        ; required when behind nginx

; ── Logging ────────────────────────────────────────────────
log_level = warn
log_handler = :WARNING,werkzeug:CRITICAL
```

### Systemd service (`/etc/systemd/system/odoo.service`)

```ini
[Unit]
Description=Odoo 18.0
After=network.target postgresql.service

[Service]
Type=simple
User=odoo
Group=odoo
ExecStart=/opt/odoo/venv/bin/python /opt/odoo/odoo/odoo-bin \
    --config=/etc/odoo/odoo.conf
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable odoo
sudo systemctl start odoo
sudo systemctl status odoo
```

---

## 3. Nginx + SSL

```nginx
# /etc/nginx/sites-available/odoo
upstream odoo { server 127.0.0.1:8069; }
upstream odoo-chat { server 127.0.0.1:8072; }

server {
    listen 80;
    server_name your-odoo.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-odoo.com;

    ssl_certificate /etc/letsencrypt/live/your-odoo.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-odoo.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    location ~* /web/static/ {
        proxy_pass http://odoo;
        expires 864000;
        add_header Cache-Control "public, immutable";
    }

    location /web/longpolling {
        proxy_pass http://odoo-chat;
        proxy_read_timeout 3600s;
    }

    location / {
        proxy_redirect off;
        proxy_pass http://odoo;
        client_max_body_size 200m;
        proxy_buffers 16 64k;
        proxy_buffer_size 128k;
    }

    location ~* /web/database/ {
        deny all;   # block DB manager on production
    }
}
```

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-odoo.com
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: odoo
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  odoo:
    image: odoo:18.0
    depends_on:
      - db
    ports:
      - "8069:8069"
      - "8072:8072"
    environment:
      HOST: db
      USER: odoo
      PASSWORD: odoo
    volumes:
      - odoo_data:/var/lib/odoo
      - ./custom-addons:/mnt/extra-addons
      - ./odoo.conf:/etc/odoo/odoo.conf
    restart: unless-stopped
    command: >
      odoo
      --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/extra-addons
      --config=/etc/odoo/odoo.conf

volumes:
  postgres_data:
  odoo_data:
```

```bash
docker compose up -d
docker compose logs -f odoo
docker compose exec odoo odoo -d your_db -u your_module --stop-after-init
```

---

## 5. Backup & Restore

```bash
#!/bin/bash
# /opt/odoo/scripts/backup.sh
set -e
DB_NAME="your_db"
BACKUP_DIR="/opt/odoo/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.tar.gz"

mkdir -p "$BACKUP_DIR"
pg_dump "$DB_NAME" | gzip > "/tmp/${DB_NAME}_${DATE}.sql.gz"
tar -czf "$BACKUP_FILE" \
    -C /opt/odoo/data/filestore "$DB_NAME" \
    -C /tmp "${DB_NAME}_${DATE}.sql.gz"
rm "/tmp/${DB_NAME}_${DATE}.sql.gz"
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
echo "Backup complete: $BACKUP_FILE"
```

```bash
# Cron — daily at 2 AM
0 2 * * * /opt/odoo/scripts/backup.sh >> /var/log/odoo/backup.log 2>&1
```

### Restore + Neutralize (for staging copies)

```bash
createdb new_db
gunzip -c your_db_backup.sql.gz | psql new_db
cp -r /opt/odoo/data/filestore/your_db /opt/odoo/data/filestore/new_db

# Neutralize — disables outgoing emails, payment providers etc.
/opt/odoo/venv/bin/python /opt/odoo/odoo/odoo-bin neutralize \
    --database new_db -c /etc/odoo/odoo.conf
```

---

## 6. Safe Production Update

```bash
# NEVER update production without these steps:

# 1. Backup first
/opt/odoo/scripts/backup.sh

# 2. Test on staging (copy of production)
createdb staging_db
pg_dump production_db | psql staging_db
./odoo-bin -d staging_db -u your_module --stop-after-init
# Fix any errors on staging before touching production

# 3. Update production
sudo systemctl stop odoo
./odoo-bin -d production_db -u your_module --stop-after-init
sudo systemctl start odoo

# 4. Verify
sudo systemctl status odoo
sudo journalctl -u odoo -n 50
```

---

## 7. Common Production Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Workers OOM | 502 Bad Gateway | Reduce `workers`, raise `limit_memory_hard` |
| Slow pages | High CPU/memory | Enable PgBouncer, add DB indexes |
| Disk full | DB crashes | Clean old logs, backups, `ir_attachment` |
| Cron silent | Scheduled actions missing | Verify `max_cron_threads > 0` |
| Module fails install | ERROR in log | Check full traceback in `odoo.log` |
| PDF blank | wkhtmltopdf error | Install patched wkhtmltopdf version |
| Sessions expire | Users log out | Check `data_dir` permissions |

### Disk cleanup for `ir_attachment`

```sql
-- Find large attachments
SELECT name, mimetype,
       pg_size_pretty(file_size::bigint),
       create_date
FROM ir_attachment
ORDER BY file_size DESC
LIMIT 20;

-- Delete old report PDFs
DELETE FROM ir_attachment
WHERE res_model = 'account.move'
  AND name LIKE '%.pdf'
  AND create_date < NOW() - INTERVAL '90 days';
```
