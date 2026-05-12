# Runbook: Connection Pool Exhaustion

**Alert covered:** PgBouncerClientsWaiting, PgBouncerClientsWaitingCritical, PgBouncerPoolNearExhaustion, PgBouncerMaxClientConnNearLimit, PgBouncerNoIdleServers, PgBouncerExporterDown

---

## Overview

Pool exhaustion means PgBouncer cannot provide server connections fast enough for the client demand. Clients queue up waiting for a connection. If unresolved, applications will experience latency spikes or timeouts (`query_wait_timeout` expiry).

PgBouncer pool exhaustion is distinct from PostgreSQL `max_connections` exhaustion:
- Pool exhaustion: PgBouncer pool is full — clients wait in PgBouncer's queue
- max_connections exhaustion: PostgreSQL itself rejects connections — immediate error

Both can be solved independently.

---

## Severity Guide

| Alert | Urgency | Likely Impact |
|---|---|---|
| PgBouncerClientsWaiting (warning) | Medium | Application latency increasing |
| PgBouncerClientsWaitingCritical (> 10) | High | Significant latency; possible timeouts |
| PgBouncerPoolNearExhaustion | Medium | Preemptive — clients will wait soon |
| PgBouncerMaxClientConnNearLimit | High | PgBouncer will reject connections |
| PgBouncerNoIdleServers | Medium | Pool saturated — next request will wait |
| PgBouncerExporterDown | High | All pool visibility lost |

---

## Step 1 — Confirm the Alert

```bash
# Check waiting clients in Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=sum(pgbouncer_pools_cl_waiting)' | \
  python3 -m json.tool | grep '"value"'

# Check directly in PgBouncer
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
```

Key values to note:
- `cl_waiting` — clients waiting (the primary indicator)
- `sv_active` — server connections in use
- `sv_idle` — server connections free
- `maxwait` — how long the longest client has been waiting

---

## Step 2 — Diagnose the Root Cause

### Is the pool too small?

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW CONFIG;" | grep pool_size
```

Compare `sv_active` from `SHOW POOLS` against `default_pool_size`. If `sv_active == default_pool_size`, the pool is full.

### Are long-running transactions holding server connections?

In transaction mode, PgBouncer releases a server connection back to the pool after each transaction. If transactions are long-running, server connections are held for longer.

```sql
-- Find long-running queries/transactions (connect to PostgreSQL, not PgBouncer)
SELECT pid, usename, datname, state,
       now() - query_start AS query_duration,
       now() - xact_start AS xact_duration,
       query
FROM pg_stat_activity
WHERE state NOT IN ('idle')
  AND (query_start < now() - interval '30 seconds'
       OR xact_start < now() - interval '30 seconds')
ORDER BY xact_duration DESC NULLS LAST
LIMIT 20;
```

### Are idle-in-transaction connections holding server connections?

```sql
SELECT pid, usename, datname, state, state_change, 
       now() - state_change AS time_in_state
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY time_in_state DESC;
```

In transaction mode, `idle in transaction` clients hold their server connection. Kill them if they are stuck.

---

## Step 3 — Immediate Remediation

### Option A — Reload PgBouncer with a larger pool size

Edit `/etc/pgbouncer/pgbouncer.ini`:
```ini
default_pool_size = 30     # increase from current value
reserve_pool_size = 10
```

Reload (no restart, no dropped connections):
```bash
sudo systemctl reload pgbouncer
```

Verify immediately:
```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
```

`sv_idle` should increase and `cl_waiting` should drop.

### Option B — Terminate stuck transactions

If long-running or idle-in-transaction sessions are holding server connections, terminate them (coordinate with the application team first):

```sql
-- Terminate idle-in-transaction sessions older than 5 minutes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < now() - interval '5 minutes';
```

### Option C — Increase max_client_conn (if hitting the total limit)

Edit `/etc/pgbouncer/pgbouncer.ini`:
```ini
max_client_conn = 300   # increase from current value
```

Reload:
```bash
sudo systemctl reload pgbouncer
```

### Option D — RECONNECT / RELOAD via PgBouncer admin

PgBouncer supports graceful operations via its admin database:

```bash
# Reload configuration without dropping connections
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "RELOAD;"

# Pause a database (stops sending new queries, drains active ones)
# Use only if you need to temporarily stop traffic to PostgreSQL
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "PAUSE mydb;"

# Resume a paused database
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "RESUME mydb;"
```

---

## Step 4 — Verify Resolution

```bash
# Check waiting clients — should be 0
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"

# Prometheus — verify waiting clients dropped
curl -s 'http://localhost:9090/api/v1/query?query=sum(pgbouncer_pools_cl_waiting)' | \
  python3 -m json.tool | grep '"value"'
```

---

## Step 5 — Handle PgBouncerExporterDown

If the alert is `PgBouncerExporterDown`:

```bash
sudo systemctl status pgbouncer-exporter
sudo systemctl start pgbouncer-exporter
sudo journalctl -u pgbouncer-exporter -n 30 --no-pager
```

If PgBouncer itself is down:

```bash
sudo systemctl status pgbouncer
sudo systemctl start pgbouncer
sudo journalctl -u pgbouncer -n 30 --no-pager
```

---

## Step 6 — Post-Incident Actions

1. Calculate appropriate `default_pool_size`:
   - Rule of thumb: `(max_connections - 10) / number_of_pools`
   - The `- 10` reserves headroom for superuser and monitoring connections
   
2. Review `query_wait_timeout` in `pgbouncer.ini`:
   - Currently set to: `psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW CONFIG;" | grep wait_timeout`
   - Recommended: 30–120 seconds depending on application tolerance
   
3. Set `idle_in_transaction_session_timeout` in PostgreSQL to auto-terminate stuck sessions:
   ```sql
   ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';
   SELECT pg_reload_conf();
   ```

4. Review application connection pool settings:
   - The application should not hold connections open longer than necessary.
   - In transaction mode: each transaction should start and commit/rollback quickly.

5. Consider adding monitoring for `pgbouncer_pools_maxwait` — sustained wait times signal chronic pool pressure even when alert thresholds are not crossed.
