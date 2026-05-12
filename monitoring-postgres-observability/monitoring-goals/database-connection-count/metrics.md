# Metrics — Database Connection Count

All metrics in this section are exposed by `postgres_exporter` and scraped by Prometheus.

---

## Core Metrics

### `pg_stat_activity_count`

The primary metric for connection count monitoring.

- **Source:** `SELECT count(*) FROM pg_stat_activity GROUP BY datname, usename, state, ...`
- **Type:** Gauge
- **Labels:** `datname`, `usename`, `state`, `wait_event_type`, `wait_event`

Example metric lines:

```
pg_stat_activity_count{datname="mydb",state="active",usename="appuser"} 5
pg_stat_activity_count{datname="mydb",state="idle",usename="appuser"} 12
pg_stat_activity_count{datname="postgres",state="active",usename="monitoring_user"} 1
```

### `pg_settings_max_connections`

The configured hard limit for total connections.

- **Source:** `SELECT setting FROM pg_settings WHERE name = 'max_connections'`
- **Type:** Gauge
- **Labels:** none (scalar)

```
pg_settings_max_connections 100
```

---

## Derived Metrics (PromQL Calculations)

These are not raw exported metrics but are computed in PromQL queries and alert rules.

| Metric Expression | What It Represents |
|---|---|
| `sum(pg_stat_activity_count)` | Total connections across all databases and users |
| `sum(pg_stat_activity_count{state="active"})` | Active query-executing connections |
| `sum(pg_stat_activity_count{state="idle"})` | Idle connections (connected, doing nothing) |
| `sum(pg_stat_activity_count{state=~"idle in transaction.*"})` | Idle-in-transaction connections (potentially holding locks) |
| `sum by (datname) (pg_stat_activity_count)` | Total connections per database |
| `sum by (usename) (pg_stat_activity_count)` | Total connections per user |
| `(sum(pg_stat_activity_count) / pg_settings_max_connections) * 100` | Connection usage as a percentage of max_connections |
| `max_over_time(sum(pg_stat_activity_count)[5m:30s])` | Peak connection count within the last 5 minutes |

---

## Metric Label Reference

| Label | Values | Notes |
|---|---|---|
| `datname` | Database name (e.g., `mydb`, `postgres`) | Can be empty for background workers |
| `usename` | PostgreSQL user (e.g., `appuser`, `monitoring_user`) | Can be empty for background workers |
| `state` | `active`, `idle`, `idle in transaction`, `idle in transaction (aborted)`, `fastpath function call`, `disabled` | Key dimension for connection analysis |
| `wait_event_type` | `Lock`, `LWLock`, `IO`, `Client`, etc. | Present only when state is `active` and waiting |
| `wait_event` | Specific wait event name | Drill-down for contention analysis |

---

## Metric Availability

These metrics are available as soon as `postgres_exporter` starts and `pg_up == 1`. No additional PostgreSQL extensions or configuration are needed — `pg_stat_activity` is always available in PostgreSQL 17.

To verify metric availability:

```bash
curl -s http://localhost:9187/metrics | grep -c "pg_stat_activity_count"
```

Expected: a number greater than 0 (one line per unique label combination).
