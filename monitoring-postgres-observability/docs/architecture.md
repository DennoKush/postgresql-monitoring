# Architecture

## Data Flow

```
PostgreSQL 17
    │
    │  pg_stat_activity, pg_stat_bgwriter, pg_settings, etc.
    ▼
postgres_exporter  (:9187)
    │
    │  HTTP GET /metrics  (Prometheus pull)
    ▼
Prometheus TSDB  ──────────────────────────────┐
    │                                          │
    │  PromQL queries                          │  Alert evaluation
    ▼                                          ▼
Grafana Dashboards                     Grafana Alerting
                                               │
                                               │  Webhook POST
                                               ▼
                                       Microsoft Teams


PgBouncer  (:6432)
    │
    │  SHOW POOLS / SHOW CLIENTS / SHOW SERVERS (internal admin DB)
    ▼
pgbouncer_exporter  (:9127)
    │
    │  HTTP GET /metrics  (Prometheus pull)
    ▼
Prometheus TSDB  (same instance — separate job)
```

---

## Component Descriptions

### PostgreSQL 17

The database server. Exposes internal state through system views:

- `pg_stat_activity` — one row per backend process, showing state, wait event, connected database, and user.
- `pg_settings` — configuration parameters including `max_connections`.
- `pg_stat_bgwriter`, `pg_stat_database` — I/O and query statistics.

PostgreSQL does **not** push metrics anywhere. A dedicated exporter queries these views on behalf of Prometheus.

### postgres_exporter

A Go binary that connects to PostgreSQL using a read-only monitoring user, queries system views, and exposes the results as Prometheus-format metrics on port **9187**.

Key metrics for this implementation:
- `pg_stat_activity_count` — connection count by state, database, user, and application.
- `pg_settings_max_connections` — the hard connection limit.

postgres_exporter is the **only** source for PostgreSQL connection count metrics in this stack.

### PgBouncer

A lightweight connection pooler. Applications connect to PgBouncer (:6432), which maintains a smaller pool of actual PostgreSQL connections and multiplexes client requests across them.

PgBouncer has an internal admin database (`pgbouncer`) accessible via the PostgreSQL wire protocol. The `SHOW` commands expose real-time pool state.

### pgbouncer_exporter

A Go binary that connects to PgBouncer's admin database, runs `SHOW POOLS`, `SHOW CLIENTS`, `SHOW SERVERS`, `SHOW STATS`, and `SHOW DATABASES`, and exposes the results as Prometheus-format metrics on port **9127**.

Key metrics for this implementation:
- `pgbouncer_pools_cl_active` — active client connections.
- `pgbouncer_pools_cl_waiting` — clients waiting for a server connection (the primary pool exhaustion signal).
- `pgbouncer_pools_sv_active` — active server (PostgreSQL) connections.
- `pgbouncer_pools_sv_idle` — idle server connections available to the pool.
- `pgbouncer_config_max_client_conn` — configured hard limit on total clients.

pgbouncer_exporter is **required** for pool exhaustion visibility. postgres_exporter cannot provide PgBouncer pool data.

### Prometheus

A time-series database that **pulls** (scrapes) metrics from exporters at a configured interval. It does not receive pushes.

Responsibilities:
- Stores all metric samples with timestamps.
- Evaluates alert rules on a configurable interval.
- Fires alerts to Alertmanager or (in Grafana-managed alerts mode) to Grafana.

In this stack, **Grafana-managed alerts** are used — Prometheus evaluates queries but Grafana owns alert state and routing.

### Grafana

Visualization and alerting front-end backed by Prometheus as a datasource.

Responsibilities:
- Dashboards displaying real-time and historical connection and pool data.
- Alert rules evaluated against Prometheus metrics.
- Contact points (Teams webhook) and notification policies.

### Microsoft Teams

Receives alert notifications from Grafana as formatted messages via an incoming webhook URL (or a Power Automate workflow webhook for newer Teams tenants).

---

## Key Architectural Distinctions

| Question | Answer |
|---|---|
| Who sources PostgreSQL connection metrics? | `postgres_exporter` reading `pg_stat_activity` |
| Who sources PgBouncer pool metrics? | `pgbouncer_exporter` reading PgBouncer SHOW commands |
| Can postgres_exporter show pool exhaustion? | No. It sees PostgreSQL-level connections only. |
| Does Prometheus push or pull? | Always pulls (scrapes). Exporters expose an HTTP endpoint. |
| Who evaluates alert rules? | Grafana (Grafana-managed alerts using Prometheus as datasource) |
| Can PostgreSQL max_connections exhaust before PgBouncer pool? | Yes. They are independent limits. Pool exhaustion is usually hit first. |
