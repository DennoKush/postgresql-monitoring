# Grafana Dashboard — PgBouncer Pool Exhaustion

Dashboard file: [`setup/grafana/dashboards/pgbouncer-pool-exhaustion.json`](../../setup/grafana/dashboards/pgbouncer-pool-exhaustion.json)

---

## Panel Descriptions

### Row 1 — Current State (Stat Panels)

| Panel | Query | Visualization | Threshold |
|---|---|---|---|
| Active Clients | `sum(pgbouncer_pools_cl_active)` | Stat | none |
| Waiting Clients | `sum(pgbouncer_pools_cl_waiting)` | Stat — red if > 0 | yellow ≥ 1, red ≥ 10 |
| Active Server Connections | `sum(pgbouncer_pools_sv_active)` | Stat | none |
| Idle Server Connections | `sum(pgbouncer_pools_sv_idle)` | Stat — red if 0 | red = 0, yellow = 1–2, green ≥ 3 |
| Pool Utilization % | `(sum(pgbouncer_pools_sv_active) / pgbouncer_config_default_pool_size) * 100` | Gauge | yellow ≥ 80%, red ≥ 95% |
| Max Client Conn Usage % | `(sum(cl_active+cl_waiting) / max_client_conn) * 100` | Gauge | yellow ≥ 75%, red ≥ 90% |

The **Waiting Clients** panel is the most important at a glance. Any non-zero value means the pool is exhausted and clients are queuing.

---

### Row 2 — Time-Series Trends

**Panel: Client Connections Over Time**
- Query A: `sum(pgbouncer_pools_cl_active)` — Active Clients
- Query B: `sum(pgbouncer_pools_cl_waiting)` — Waiting Clients (colored red)
- Visualization: Time series — overlaid
- Purpose: Shows when pool exhaustion began and how long it lasted.

**Panel: Server Connections Over Time**
- Query A: `sum(pgbouncer_pools_sv_active)` — Active Servers
- Query B: `sum(pgbouncer_pools_sv_idle)` — Idle Servers
- Visualization: Time series — overlaid
- Purpose: Idle servers dropping to zero correlates with waiting clients appearing.

---

### Row 3 — Distribution Tables

**Panel: Pool Usage by Database**
- Query A: `sum by (database) (pgbouncer_pools_cl_active)` — Active clients per DB
- Query B: `sum by (database) (pgbouncer_pools_cl_waiting)` — Waiting clients per DB
- Visualization: Table
- Purpose: Identify which database is under pool pressure.

**Panel: Pool Usage by User**
- Query A: `sum by (user) (pgbouncer_pools_cl_active)` — Active clients per user
- Query B: `sum by (user) (pgbouncer_pools_cl_waiting)` — Waiting clients per user
- Visualization: Table
- Purpose: Identify which user/application is exhausting the pool.

---

### Row 4 — Health

**Panel: pgbouncer_exporter Health**
- Query: `pgbouncer_up`
- Visualization: Stat — green if 1, red if 0
- Purpose: Single-pane confirmation that pool metrics are live.

---

## Drill-Down Queries (Explore)

When `cl_waiting > 0`, use these in Grafana Explore to diagnose:

```promql
# Which database has waiting clients?
sum by (database) (pgbouncer_pools_cl_waiting)

# Which user has waiting clients?
sum by (user) (pgbouncer_pools_cl_waiting)

# How long has the pool been saturated?
max_over_time(sum(pgbouncer_pools_cl_waiting)[1h:1m])

# Is there a pattern — time of day?
sum(pgbouncer_pools_cl_waiting)
```

Also connect directly to PgBouncer admin DB for real-time details:
```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW CLIENTS;"
```

---

## Import Instructions

1. Grafana → **Dashboards → Import**
2. Upload `setup/grafana/dashboards/pgbouncer-pool-exhaustion.json`
3. Select Prometheus datasource
4. Click **Import**

---

## Suggested Alert Integration

Add an **Alert list** panel showing alerts from the `connection-pool-exhaustion` folder. This surfaces `PgBouncerClientsWaiting` and related alerts alongside the pool metrics panels.
