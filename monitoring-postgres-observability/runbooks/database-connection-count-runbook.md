# Runbook: Database Connection Count

**Alert covered:** HighDatabaseConnectionUsage, DatabaseConnectionsNearMax, TooManyIdleConnections, SuddenConnectionSpike

---

## Overview

This runbook guides the on-call engineer through diagnosing and resolving high PostgreSQL connection count alerts. The goal is to bring connection usage below 70% of `max_connections` without impacting running workloads.

---

## Severity Guide

| Alert | Urgency | Likely Impact |
|---|---|---|
| HighDatabaseConnectionUsage (> 70%) | Medium | Reduced headroom; risk of future failures |
| DatabaseConnectionsNearMax (> 90%) | High — act immediately | New connections will fail at 100% |
| TooManyIdleConnections | Low | Memory pressure; potential leak |
| SuddenConnectionSpike | Medium | Investigate source; may escalate |

---

## Step 1 — Confirm the Alert

```bash
# Connect to Prometheus — verify current value
curl -s 'http://localhost:9090/api/v1/query?query=(sum(pg_stat_activity_count)/pg_settings_max_connections)*100' | \
  python3 -m json.tool | grep '"value"'

# Check current max_connections
curl -s 'http://localhost:9090/api/v1/query?query=pg_settings_max_connections' | \
  python3 -m json.tool | grep '"value"'
```

---

## Step 2 — Identify the Source

### By database

```sql
SELECT datname, count(*)
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;
```

### By user

```sql
SELECT usename, count(*)
FROM pg_stat_activity
GROUP BY usename
ORDER BY count(*) DESC;
```

### By state

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

### Look for idle-in-transaction sessions

```sql
SELECT pid, usename, datname, state, query_start, 
       now() - state_change AS time_in_state, query
FROM pg_stat_activity
WHERE state IN ('idle in transaction', 'idle in transaction (aborted)')
ORDER BY time_in_state DESC;
```

---

## Step 3 — Remediation Options

### Option A — Terminate idle connections (non-destructive)

Terminate connections that have been idle for more than 5 minutes:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND state_change < now() - interval '5 minutes'
  AND pid <> pg_backend_pid()
  AND usename <> 'monitoring_user';
```

Verify the count dropped:

```sql
SELECT count(*) FROM pg_stat_activity WHERE state = 'idle';
```

### Option B — Terminate idle-in-transaction connections

These hold locks. Terminate only after confirming with the application team:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < now() - interval '10 minutes';
```

### Option C — Increase max_connections (last resort, requires restart)

Only if the workload genuinely requires more connections and other options are exhausted:

1. Edit `/etc/postgresql/18/main/postgresql.conf`
2. Increase `max_connections` (also consider increasing `shared_buffers` if memory allows)
3. Plan a maintenance window for the restart

**Note:** Increasing `max_connections` requires a PostgreSQL restart, not just a reload. Schedule accordingly.

### Option D — Reduce PgBouncer pool size

If `postgres_exporter` shows many connections from the PgBouncer user, the pool size may be too large relative to `max_connections`:

1. Calculate: `max_connections` × 0.8 / number of PgBouncer pools = recommended per-pool size
2. Edit `/etc/pgbouncer/pgbouncer.ini`, reduce `default_pool_size`
3. Reload: `sudo systemctl reload pgbouncer`

---

## Step 4 — Verify Resolution

```sql
SELECT 
    count(*) AS current_connections,
    setting::int AS max_connections,
    round((count(*)::numeric / setting::int) * 100, 2) AS usage_percent
FROM pg_stat_activity, pg_settings
WHERE name = 'max_connections'
GROUP BY setting;
```

Confirm `usage_percent` is below 70%.

In Prometheus:
```bash
curl -s 'http://localhost:9090/api/v1/query?query=(sum(pg_stat_activity_count)/pg_settings_max_connections)*100' | \
  python3 -m json.tool | grep '"value"'
```

---

## Step 5 — Post-Incident Actions

1. Identify root cause (deployment, connection leak, misconfigured pool).
2. If a connection leak, file a ticket with the application team to fix the pool configuration.
3. Review PgBouncer `default_pool_size` against `max_connections` — ensure total pool capacity does not exceed 80% of `max_connections`.
4. Consider setting `idle_in_transaction_session_timeout = 60s` in `postgresql.conf` to auto-terminate stuck transactions.
5. Consider setting `client_idle_timeout` in PgBouncer to reclaim idle connections.

---

## Configuration Commands Reference

```sql
-- Check current timeout settings
SHOW idle_in_transaction_session_timeout;
SHOW statement_timeout;

-- Set idle-in-transaction timeout (run as superuser)
ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';
SELECT pg_reload_conf();

-- Verify
SHOW idle_in_transaction_session_timeout;
```
