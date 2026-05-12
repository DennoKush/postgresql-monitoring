# Alert Rules — Database Connection Count

All alert rules are defined in:  
[`setup/prometheus/rules/postgres-connection-count.yml`](../../setup/prometheus/rules/postgres-connection-count.yml)

---

## Alert Summary

| Alert Name | Condition | For | Severity |
|---|---|---|---|
| HighDatabaseConnectionUsage | Connection usage > 70% of max_connections | 5m | warning |
| DatabaseConnectionsNearMax | Connection usage > 90% of max_connections | 2m | critical |
| TooManyIdleConnections | Idle connections > 3× active connections | 10m | warning |
| SuddenConnectionSpike | Connection count grows by > 30 in 5 minutes | 2m | warning |

---

## Alert Details

### HighDatabaseConnectionUsage

**Purpose:** Early warning before connection exhaustion becomes critical.

**Expression:**
```yaml
expr: |
  (
    sum(pg_stat_activity_count) by (instance)
    /
    pg_settings_max_connections
  ) * 100 > 70
```

**Tuning guidance:**
- Lower the threshold (e.g., 60%) if you want earlier warning.
- Raise it (e.g., 80%) if your workload routinely runs high and false positives are a problem.
- The `for: 5m` prevents transient spikes from paging — remove or reduce it if you need immediate notification.

**Recommended action:**
1. Check which database or user is driving the high count.
2. Identify and kill long-running idle or idle-in-transaction connections.
3. Review PgBouncer `default_pool_size` — if pools are too large relative to `max_connections`, reduce them.

---

### DatabaseConnectionsNearMax

**Purpose:** Critical — action required before new connections are rejected.

**Expression:**
```yaml
expr: |
  (
    sum(pg_stat_activity_count) by (instance)
    /
    pg_settings_max_connections
  ) * 100 > 90
```

**Tuning guidance:**
- `for: 2m` is aggressive intentionally — at 90%, the system is 10% away from total failure.
- On very small `max_connections` (e.g., 50), even 5 connections above 90% = 45+ connections.

**Recommended action:**
1. Immediate: identify and kill idle/idle-in-transaction connections:
   ```sql
   SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity
   WHERE state = 'idle'
     AND state_change < now() - interval '5 minutes'
     AND pid <> pg_backend_pid();
   ```
2. Check for connection leaks in the application.
3. Temporary: if safe, increase `max_connections` (requires PostgreSQL restart).

---

### TooManyIdleConnections

**Purpose:** Detect connection leaks or oversized pools before they contribute to exhaustion.

**Expression:**
```yaml
expr: |
  sum(pg_stat_activity_count{state="idle"}) by (instance)
  >
  sum(pg_stat_activity_count{state="active"}) by (instance) * 3
```

**Tuning guidance:**
- The `3×` multiplier is conservative. Adjust to `2×` for stricter enforcement.
- Only fires when there is at least some active traffic (denominator > 0).
- `for: 10m` avoids alerts during startup surges.

**Recommended action:**
1. Check `idle_in_transaction_session_timeout` — set it if not configured.
2. Review application connection pool configuration.
3. Check for connection leaks (connections never returned to the pool).

---

### SuddenConnectionSpike

**Purpose:** Detect connection storms (retry loops, deployment floods, misconfigured pools).

**Expression:**
```yaml
expr: |
  (
    sum(pg_stat_activity_count) by (instance)
    -
    sum(pg_stat_activity_count) by (instance) offset 5m
  ) > 30
```

**Tuning guidance:**
- The `30` threshold is appropriate for environments with `max_connections` in the 100–200 range.
- For smaller environments (max_connections ≤ 50), lower the threshold to 10–15.
- For large environments (max_connections ≥ 500), raise it to 50–100.

**Recommended action:**
1. Check if a deployment or application restart just occurred.
2. Verify connections are stabilizing — if they keep climbing, investigate the source.
3. If a retry storm is suspected, temporarily reject new connections at the load balancer while the root cause is addressed.

---

## Adjusting Alert Labels

These labels are used for notification routing in Grafana:

```yaml
labels:
  severity: warning   # or critical
  team: dba
  component: postgres
```

Adjust `team` and add custom labels to match your organization's routing rules in Grafana notification policies.
