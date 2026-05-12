# Monitoring Goal: Database Connection Count

## What Is Database Connection Count?

Every client connecting to PostgreSQL consumes a backend process on the server. PostgreSQL allocates memory (approximately 5–10 MB per connection) and operating system resources for each. The total number of open connections at any moment is the database connection count.

PostgreSQL enforces a hard ceiling via the `max_connections` parameter. When this ceiling is reached, the next connection attempt fails immediately with:

```
FATAL: sorry, too many clients already
```

This is a hard failure — the application receives an error and cannot connect until an existing connection closes.

---

## Why It Matters

| Risk | Consequence |
|---|---|
| `max_connections` exhausted | New connections rejected; application errors |
| Too many idle connections | Memory waste; autovacuum slot contention |
| Connection spike (storm) | Rapid exhaustion risk; server overload |
| Long-running idle-in-transaction | Holds locks; blocks autovacuum; inflates counts |

---

## Connection States

PostgreSQL connections appear in `pg_stat_activity` with the following states:

| State | Meaning |
|---|---|
| `active` | Currently executing a query |
| `idle` | Connected but doing nothing |
| `idle in transaction` | Inside a transaction block, not currently executing |
| `idle in transaction (aborted)` | In a failed transaction, not yet rolled back |
| `fastpath function call` | Executing a fast-path function |
| `disabled` | `track_activities` disabled for this session |

**`idle` connections** hold server resources without doing useful work. A high idle-to-active ratio often indicates a connection pool that is too large, or connection leaks in the application.

**`idle in transaction`** connections are particularly dangerous — they hold row-level locks and block autovacuum.

---

## Signal Source

| Signal | Source |
|---|---|
| Connection count by state | `postgres_exporter` reading `pg_stat_activity` |
| Connection count by database | `postgres_exporter` reading `pg_stat_activity` |
| Connection count by user | `postgres_exporter` reading `pg_stat_activity` |
| `max_connections` setting | `postgres_exporter` reading `pg_settings` |
| Connection usage percentage | Calculated in PromQL from the above two |

`postgres_exporter` is the primary source. It queries these views on behalf of Prometheus using the read-only `monitoring_user`.

---

## Manual Validation with SQL

Connect as the monitoring user and run these queries to verify connection state before or alongside Prometheus data.

### Connections by State

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

### Connections by Database

```sql
SELECT datname, count(*)
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;
```

### Connections by User

```sql
SELECT usename, count(*)
FROM pg_stat_activity
GROUP BY usename
ORDER BY count(*) DESC;
```

### max_connections Setting

```sql
SHOW max_connections;
```

### Connection Usage Percentage

```sql
SELECT
    count(*) AS current_connections,
    setting::int AS max_connections,
    round((count(*)::numeric / setting::int) * 100, 2) AS usage_percent
FROM pg_stat_activity, pg_settings
WHERE name = 'max_connections'
GROUP BY setting;
```

### Idle in Transaction (potential lock holders)

```sql
SELECT pid, usename, datname, state, query_start, state_change, query
FROM pg_stat_activity
WHERE state IN ('idle in transaction', 'idle in transaction (aborted)')
ORDER BY state_change;
```

---

## Grafana Validation

After Prometheus is scraping `postgres_exporter`, run these checks in Grafana's Explore view or on the dashboard.

1. Open Grafana → **Explore** → select Prometheus datasource.
2. Run the query: `sum(pg_stat_activity_count)` — should return the current total connection count.
3. Confirm the value matches the SQL result from `SELECT count(*) FROM pg_stat_activity`.

---

## Alert Thresholds

| Alert | Threshold | Severity | For |
|---|---|---|---|
| HighDatabaseConnectionUsage | > 70% of max_connections | warning | 5m |
| DatabaseConnectionsNearMax | > 90% of max_connections | critical | 2m |
| TooManyIdleConnections | idle > 3× active | warning | 10m |
| SuddenConnectionSpike | +30 connections in 5m | warning | 2m |

Adjust thresholds based on your `max_connections` setting and typical workload.

---

## How to Visualize in Grafana

Use the **PostgreSQL — Connection Count** dashboard in `setup/grafana/dashboards/postgres-connection-count.json`.

Key panels:
1. **Total Active Connections** — stat panel, current value
2. **Connection Usage %** — gauge with color thresholds at 70% (yellow) and 90% (red)
3. **max_connections** — reference value stat panel
4. **Connection Count by State** — time-series showing active vs idle trend
5. **Connection Usage % Over Time** — time-series with threshold lines
6. **Connections by Database** — table ranked by count
7. **Connections by User** — table ranked by count
8. **Peak Connection Count (5m window)** — `max_over_time` to catch spikes

---

## Troubleshooting

See [`troubleshooting.md`](troubleshooting.md) for detailed steps.

Quick checks:

```bash
# Is postgres_exporter running and pg_up = 1?
curl -s http://localhost:9187/metrics | grep pg_up

# Are connection count metrics present?
curl -s http://localhost:9187/metrics | grep pg_stat_activity_count | head -10

# Is Prometheus scraping successfully?
curl -s 'http://localhost:9090/api/v1/query?query=sum(pg_stat_activity_count)' | python3 -m json.tool
```
