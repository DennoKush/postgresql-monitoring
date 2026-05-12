# Alert Rules — Connection Pool Exhaustion

All alert rules are defined in:  
[`setup/prometheus/rules/pgbouncer-pool-exhaustion.yml`](../../setup/prometheus/rules/pgbouncer-pool-exhaustion.yml)

---

## Alert Summary

| Alert Name | Condition | For | Severity |
|---|---|---|---|
| PgBouncerClientsWaiting | `cl_waiting > 0` | 1m | warning |
| PgBouncerClientsWaitingCritical | `cl_waiting > 10` | 2m | critical |
| PgBouncerPoolNearExhaustion | `sv_active / pool_size > 80%` | 5m | warning |
| PgBouncerMaxClientConnNearLimit | `total_clients / max_client_conn > 85%` | 5m | critical |
| PgBouncerNoIdleServers | `cl_active > 0 AND sv_idle == 0` | 3m | warning |
| PgBouncerExporterDown | `pgbouncer_up == 0 or absent` | 2m | critical |

---

## Alert Details

### PgBouncerClientsWaiting

**Purpose:** Detect pool exhaustion as soon as clients start waiting. This is the earliest signal.

**Expression:**
```yaml
expr: sum(pgbouncer_pools_cl_waiting) by (instance) > 0
for: 1m
```

**Why `for: 1m`:** Very brief waits (sub-second) happen during normal operation. The `for: 1m` requires sustained waiting before alerting.

**Tuning guidance:**
- For latency-sensitive applications, remove or reduce `for` to alert immediately.
- Add `by (database, user)` to the expression if you want per-pool granularity.

**Recommended action:**
1. Connect to PgBouncer admin DB and run `SHOW POOLS` to see which pool is affected.
2. Check how many server connections are active vs idle.
3. If all server connections are active — the pool is too small for the current load.
4. Temporarily increase `default_pool_size` (live reload: `RELOAD;` in PgBouncer admin DB, or `sudo systemctl reload pgbouncer`).

---

### PgBouncerClientsWaitingCritical

**Purpose:** High number of waiting clients — application is experiencing significant latency.

**Expression:**
```yaml
expr: sum(pgbouncer_pools_cl_waiting) by (instance) > 10
for: 2m
```

**Recommended action:**
1. Same as above, but with urgency.
2. If `query_wait_timeout` is set, applications may already be receiving errors.
3. Check for long-running transactions holding server connections:
   ```sql
   SELECT pid, usename, datname, state, query_start, now() - query_start AS duration, query
   FROM pg_stat_activity
   WHERE state != 'idle'
   ORDER BY duration DESC;
   ```
4. Kill stuck transactions if appropriate.

---

### PgBouncerPoolNearExhaustion

**Purpose:** Early warning — server connections approaching pool limit. Fires before clients actually wait.

**Expression:**
```yaml
expr: |
  (
    sum(pgbouncer_pools_sv_active) by (instance, database, user)
    /
    pgbouncer_config_default_pool_size
  ) * 100 > 80
for: 5m
```

**Tuning guidance:**
- The 80% threshold provides a buffer. Lower to 70% for an earlier warning.
- `for: 5m` avoids alerting on burst traffic that resolves itself.

**Recommended action:**
1. Investigate which workload is consuming server connections.
2. Check if long-running transactions are blocking pool turnover.
3. Plan a `default_pool_size` increase if this is a recurring pattern.

---

### PgBouncerMaxClientConnNearLimit

**Purpose:** Total client connections approaching `max_client_conn`. If this is reached, PgBouncer will reject new connections entirely.

**Expression:**
```yaml
expr: |
  (
    sum(pgbouncer_pools_cl_active + pgbouncer_pools_cl_waiting) by (instance)
    /
    pgbouncer_config_max_client_conn
  ) * 100 > 85
for: 5m
```

**Recommended action:**
1. Confirm `max_client_conn` value: `SHOW CONFIG;` in PgBouncer admin.
2. Increase `max_client_conn` in `pgbouncer.ini` and reload: `sudo systemctl reload pgbouncer`.
3. This is separate from pool size — `max_client_conn` is the absolute ceiling for all clients, across all pools.

---

### PgBouncerNoIdleServers

**Purpose:** Pool is saturated — no server connections are free. New active clients will have to wait immediately.

**Expression:**
```yaml
expr: |
  (sum(pgbouncer_pools_cl_active) by (instance, database, user) > 0)
  and
  (sum(pgbouncer_pools_sv_idle) by (instance, database, user) == 0)
for: 3m
```

**Recommended action:**
1. Check `SHOW POOLS` — are all `sv_active` connections tied to long transactions?
2. Check `SHOW CLIENTS` for idle-in-transaction clients holding server connections.
3. In transaction mode, server connections should be released after each transaction. If they are not being released, suspect idle-in-transaction application sessions.

---

### PgBouncerExporterDown

**Purpose:** If the exporter is down, all pool visibility is lost. This is a critical blind spot.

**Expression:**
```yaml
expr: pgbouncer_up == 0 or absent(pgbouncer_up)
for: 2m
```

**Recommended action:**
```bash
sudo systemctl status pgbouncer-exporter
sudo systemctl start pgbouncer-exporter
sudo journalctl -u pgbouncer-exporter -n 30 --no-pager
```

---

## Alert Label Reference

These labels route alerts in Grafana:

```yaml
labels:
  severity: warning   # or critical
  team: dba
  component: pgbouncer
```

Adjust `team` to match your Grafana notification policy matchers.
