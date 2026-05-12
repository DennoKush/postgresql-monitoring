# postgres_exporter Validation

Run these checks after completing the installation steps in `README.md`.

---

## 1. Service Status

```bash
sudo systemctl status postgres_exporter
```

Expected output includes `Active: active (running)`.

---

## 2. Metrics Endpoint

```bash
curl -s http://localhost:9187/metrics | grep -E "^pg_up|^pg_stat_activity_count"
```

Expected:
```
pg_up 1
pg_stat_activity_count{...} <number>
```

`pg_up 0` means postgres_exporter is running but cannot connect to PostgreSQL. Check credentials and `pg_hba.conf`.

---

## 3. PostgreSQL Connection Count Metrics Present

```bash
curl -s http://localhost:9187/metrics | grep pg_stat_activity_count | head -20
```

You should see multiple lines with labels like `{datname="postgres", state="idle", ...}`.

---

## 4. max_connections Metric Present

```bash
curl -s http://localhost:9187/metrics | grep pg_settings_max_connections
```

Expected (example):
```
pg_settings_max_connections 100
```

---

## 5. Monitoring User Access

Verify the monitoring user can execute the underlying queries directly:

```bash
psql -h 127.0.0.1 -U monitoring_user -d postgres -c "
  SELECT state, count(*)
  FROM pg_stat_activity
  GROUP BY state
  ORDER BY count(*) DESC;"
```

Expected: returns rows (at least one row for the current connection).

---

## 6. Firewall / Remote Reachability

Run from the **Observability Server**:

```bash
curl -s http://<PG17_HOST_IP>:9187/metrics | grep pg_up
```

Expected: `pg_up 1`

If this fails but the local test passes, check firewall rules — see [`docs/ports-and-networking.md`](../../docs/ports-and-networking.md).

---

## 7. Logs

```bash
sudo journalctl -u postgres_exporter -n 30 --no-pager
```

Look for any ERROR or FATAL lines.

---

## Validation Checklist

- [ ] `systemctl status postgres_exporter` shows `active (running)`
- [ ] `pg_up 1` returned from `/metrics`
- [ ] `pg_stat_activity_count` metrics present with expected labels
- [ ] `pg_settings_max_connections` metric present
- [ ] Monitoring user can query `pg_stat_activity` directly
- [ ] Metrics endpoint reachable from Observability Server
