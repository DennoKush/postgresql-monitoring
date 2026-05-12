# pgbouncer_exporter Setup

pgbouncer_exporter connects to PgBouncer's admin database and exposes pool metrics on port **9127** for Prometheus to scrape.

This exporter is the **only** source for pool exhaustion metrics in this stack. `postgres_exporter` cannot provide PgBouncer pool data.

---

## Prerequisites

- PgBouncer is installed and running (see [`setup/pgbouncer/README.md`](../pgbouncer/README.md))
- PgBouncer's admin database is reachable:
  ```bash
  psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW VERSION;"
  ```

---

## 1. Download and Install the Binary

```bash
# Download latest release (adjust version as needed)
wget https://github.com/prometheus-community/pgbouncer_exporter/releases/download/v0.7.0/pgbouncer_exporter-0.7.0.linux-amd64.tar.gz

# Verify checksum
sha256sum pgbouncer_exporter-0.7.0.linux-amd64.tar.gz

# Extract
tar -xzf pgbouncer_exporter-0.7.0.linux-amd64.tar.gz

# Install binary
sudo cp pgbouncer_exporter-0.7.0.linux-amd64/pgbouncer_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/pgbouncer_exporter

# Verify
/usr/local/bin/pgbouncer_exporter --version
```

---

## 2. Create Service User and Directories

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin pgb_exporter
sudo mkdir -p /etc/pgbouncer-exporter
```

---

## 3. Configure the Environment File

```bash
sudo cp /path/to/this/repo/setup/pgbouncer-exporter/.env.example /etc/pgbouncer-exporter/.env
sudo nano /etc/pgbouncer-exporter/.env
# Fill in the actual password for pgb_exporter
sudo chmod 600 /etc/pgbouncer-exporter/.env
sudo chown pgb_exporter:pgb_exporter /etc/pgbouncer-exporter/.env
```

The `DATA_SOURCE_NAME` format for pgbouncer_exporter:

```
postgresql://pgb_exporter:<password>@localhost:6432/pgbouncer?sslmode=disable
```

Note: the database name is literally `pgbouncer` — this is PgBouncer's internal admin database.

---

## 4. Install the systemd Service

```bash
sudo cp /path/to/this/repo/setup/pgbouncer-exporter/pgbouncer-exporter.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable pgbouncer-exporter
sudo systemctl start pgbouncer-exporter
```

---

## 5. Validate

See [`validation.md`](validation.md).

Quick check:

```bash
sudo systemctl status pgbouncer-exporter
curl -s http://localhost:9127/metrics | grep pgbouncer_up
# Expected: pgbouncer_up 1
```

---

## Troubleshooting

### pgbouncer_exporter not starting

```bash
sudo journalctl -u pgbouncer-exporter -n 50 --no-pager
```

### pgbouncer_exporter metric names not matching

Different versions of pgbouncer_exporter use different metric name prefixes. The 0.7.x series uses `pgbouncer_` prefix.

Verify:
```bash
curl -s http://localhost:9127/metrics | grep -E "^pgbouncer_pools" | head -5
```

If you see `pgbouncer2_` or no `pgbouncer_pools` metrics, you may have a different exporter version. Adjust PromQL queries and alert rules accordingly.

### PgBouncer pool metrics missing

**Symptom:** `pgbouncer_up 1` but no `pgbouncer_pools_*` metrics.

**Fix:** Verify PgBouncer has active pools by running:
```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
```
If only the `pgbouncer` admin pool shows, no application pools exist yet. They appear only after first connection through PgBouncer.

### Wrong listen_address

**Symptom:** Prometheus can't reach port 9127.

**Fix:** Check `--web.listen-address` in the service file. Use `0.0.0.0:9127` when Prometheus is remote.

### PgBouncer admin database not reachable from exporter

**Symptom:** `pgbouncer_up 0`

**Fix:**
1. Verify PgBouncer is running: `systemctl status pgbouncer`
2. Verify password in `/etc/pgbouncer-exporter/.env` matches `userlist.txt`
3. Verify `pgb_exporter` is listed in `stats_users` in `pgbouncer.ini`
4. Test manually:
   ```bash
   psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
   ```
