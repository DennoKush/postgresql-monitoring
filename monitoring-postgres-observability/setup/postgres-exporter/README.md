# postgres_exporter Setup

postgres_exporter connects to PostgreSQL as a read-only monitoring user and exposes metrics on port **9187** for Prometheus to scrape.

---

## 1. Create the PostgreSQL Monitoring User

Run these commands as the `postgres` superuser:

```bash
sudo -u postgres psql
```

```sql
-- Create the monitoring user with a strong password
CREATE USER monitoring_user WITH PASSWORD '<strong_password_here>';

-- Grant the pg_monitor role for read-only access to all system views
GRANT pg_monitor TO monitoring_user;

-- Verify the grant
\du monitoring_user
```

### pg_hba.conf entry

Add this line to `/etc/postgresql/17/main/pg_hba.conf` to allow the monitoring user to connect from localhost:

```
host    postgres        monitoring_user     127.0.0.1/32    scram-sha-256
```

Then reload PostgreSQL:

```bash
sudo systemctl reload postgresql
```

Verify the user can connect:

```bash
psql -h 127.0.0.1 -U monitoring_user -d postgres -c "SELECT current_user, pg_is_in_recovery();"
```

---

## 2. Download and Install the Binary

```bash
# Download the latest release (adjust version as needed)
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz

# Verify checksum (find SHA256SUMS on the releases page)
sha256sum postgres_exporter-0.15.0.linux-amd64.tar.gz

# Extract
tar -xzf postgres_exporter-0.15.0.linux-amd64.tar.gz

# Install binary
sudo cp postgres_exporter-0.15.0.linux-amd64/postgres_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/postgres_exporter

# Verify
/usr/local/bin/postgres_exporter --version
```

---

## 3. Create Service User and Directories

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin postgres_exporter
sudo mkdir -p /etc/postgres_exporter
```

---

## 4. Configure the Environment File

```bash
sudo cp /path/to/this/repo/setup/postgres-exporter/.env.example /etc/postgres_exporter/.env
sudo nano /etc/postgres_exporter/.env
# Fill in the actual password
sudo chmod 600 /etc/postgres_exporter/.env
sudo chown postgres_exporter:postgres_exporter /etc/postgres_exporter/.env
```

The `DATA_SOURCE_NAME` format used by postgres_exporter:

```
postgresql://monitoring_user:<password>@localhost:5432/postgres?sslmode=disable
```

---

## 5. Install the systemd Service

```bash
sudo cp /path/to/this/repo/setup/postgres-exporter/postgres-exporter.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable postgres_exporter
sudo systemctl start postgres_exporter
```

---

## 6. Validate

See [`validation.md`](validation.md) for full validation steps.

Quick check:

```bash
sudo systemctl status postgres_exporter
curl -s http://localhost:9187/metrics | grep pg_up
# Expected: pg_up 1
```

---

## Troubleshooting

### postgres_exporter fails to start

```bash
sudo journalctl -u postgres_exporter -n 50 --no-pager
```

### Authentication failure

**Symptom:** `pg_up 0` or `FATAL: password authentication failed for user "monitoring_user"`

**Fix:**
1. Verify `pg_hba.conf` has the correct entry and PostgreSQL was reloaded.
2. Verify the password in `.env` matches the one set in PostgreSQL.
3. Test connectivity manually:
   ```bash
   psql -h 127.0.0.1 -U monitoring_user -d postgres
   ```

### pg_hba.conf blocking exporter

**Symptom:** `pg_up 0` and `FATAL: no pg_hba.conf entry for host "127.0.0.1"`

**Fix:**
1. Open `/etc/postgresql/17/main/pg_hba.conf`.
2. Confirm the entry uses `127.0.0.1/32` (not `::1/128`) if connecting over IPv4.
3. Reload: `sudo systemctl reload postgresql`

### Exporter listening on localhost only

**Symptom:** Prometheus target shows DOWN even though exporter appears running locally.

**Fix:** Check the `--web.listen-address` flag in the service file. Change `127.0.0.1:9187` to `0.0.0.0:9187` if Prometheus is on a different host.

### Incorrect PostgreSQL 17 config path

The config directory for PostgreSQL 17 on Ubuntu 24.04 is:
```
/etc/postgresql/17/main/
```
Not `/etc/postgresql/` or `/var/lib/postgresql/`. Verify with:
```bash
sudo -u postgres psql -c "SHOW config_file;"
```
