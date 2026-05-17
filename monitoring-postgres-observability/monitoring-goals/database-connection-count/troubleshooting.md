# Troubleshooting — Database Connection Count

---

## postgres_exporter Not Starting

**Symptom:** `systemctl status postgres_exporter` shows `failed` or `activating`.

**Diagnosis:**
```bash
sudo journalctl -u postgres_exporter -n 50 --no-pager
```

**Common causes and fixes:**

| Log message | Fix |
|---|---|
| `cannot load TLS configuration` | Check SSL settings in DATA_SOURCE_NAME; try adding `?sslmode=disable` |
| `no such file or directory: /usr/local/bin/postgres_exporter` | Binary not installed — re-run installation step |
| `permission denied` on `.env` file | `sudo chown postgres_exporter /etc/postgres_exporter/.env` |
| `invalid DSN: missing password` | Edit `.env` and set the correct password |

---

## PostgreSQL Authentication Failure

**Symptom:** `pg_up 0` in exporter metrics.

**Diagnosis:**
```bash
# Test connection directly
psql -h 127.0.0.1 -U monitoring_user -d postgres -c "SELECT 1;"

# Check exporter logs for auth error
sudo journalctl -u postgres_exporter -n 20 --no-pager | grep -i "fatal\|error\|auth"
```

**Fix:**
1. Verify the password in `/etc/postgres_exporter/.env` matches the one set in PostgreSQL.
2. Verify `pg_hba.conf` has the correct entry:
   ```
   host    postgres    monitoring_user    127.0.0.1/32    scram-sha-256
   ```
3. Reload PostgreSQL after any `pg_hba.conf` change:
   ```bash
   sudo systemctl reload postgresql
   ```

---

## pg_hba.conf Blocking the Exporter

**Symptom:** `FATAL: no pg_hba.conf entry for host "127.0.0.1", user "monitoring_user"`

**Fix:**
1. Open `/etc/postgresql/18/main/pg_hba.conf`.
2. Add (or verify) the line:
   ```
   host    postgres    monitoring_user    127.0.0.1/32    scram-sha-256
   ```
3. If postgres_exporter connects over IPv6 (::1), add a separate line for IPv6.
4. Reload: `sudo systemctl reload postgresql`
5. Verify: `psql -h 127.0.0.1 -U monitoring_user -d postgres -c "SELECT 1;"`

---

## Prometheus Target Down

**Symptom:** `postgres_exporter` target shows **DOWN** in Prometheus targets page.

**Step 1 — Check what Prometheus reports:**
```bash
curl -s 'http://localhost:9090/api/v1/targets' | \
  python3 -c "import json,sys; data=json.load(sys.stdin); \
  [print(t['labels']['job'], t['lastError']) for t in data['data']['activeTargets'] if t['health']=='down']"
```

**Step 2 — Verify network:**
```bash
curl -s http://<PG18_HOST_IP>:9187/metrics | head -3
```

**Step 3 — Verify exporter listen address:**
```bash
# On PG18 Host
ss -tlnp | grep 9187
```
If it shows `127.0.0.1:9187`, change to `0.0.0.0:9187` in the service file and restart.

**Step 4 — Check firewall:**
```bash
sudo ufw status verbose | grep 9187
```

---

## pg_stat_activity_count Missing or Zero

**Symptom:** `pg_up 1` but no `pg_stat_activity_count` metrics, or all values are 0.

**Diagnosis:**
```bash
curl -s http://localhost:9187/metrics | grep -c pg_stat_activity
# Expected: > 0
```

**Fix:**
1. Verify the monitoring user has `pg_monitor` role:
   ```sql
   SELECT rolname, pg_has_role('monitoring_user', rolname, 'member') 
   FROM pg_roles WHERE rolname = 'pg_monitor';
   ```
2. Verify `pg_stat_activity` is accessible:
   ```bash
   psql -h 127.0.0.1 -U monitoring_user -d postgres \
     -c "SELECT count(*) FROM pg_stat_activity;"
   ```

---

## Dashboard Shows No Data

**Symptom:** Grafana dashboard panels show "No data".

**Checks:**
1. Verify datasource is connected: Grafana → Connections → Prometheus → **Save & Test**
2. Verify the query works in Explore: `sum(pg_stat_activity_count)`
3. Check time range — ensure it covers the period after deployment
4. Verify Prometheus is running and targets are UP

---

## Wrong PostgreSQL 18 Config Path

On Ubuntu 24.04, PostgreSQL 18 configuration is at:

```
/etc/postgresql/18/main/postgresql.conf
/etc/postgresql/18/main/pg_hba.conf
```

Not `/etc/postgresql/` or `/etc/postgresql/14/` (if you have multiple versions).

Confirm the active config file:
```bash
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
```
