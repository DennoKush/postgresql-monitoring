# PromQL Queries — Connection Pool Exhaustion

Use these queries in the Grafana Explore view, dashboard panels, or alert rules.

---

## Waiting Clients (Primary Exhaustion Signal)

```promql
# Total waiting clients across all pools
sum(pgbouncer_pools_cl_waiting)

# Waiting clients per database
sum by (database) (pgbouncer_pools_cl_waiting)

# Waiting clients per user
sum by (user) (pgbouncer_pools_cl_waiting)

# Maximum wait time across all pools (seconds)
max(pgbouncer_pools_maxwait)
```

`cl_waiting > 0` is the definitive signal that pool exhaustion is actively occurring.

---

## Active vs Idle Server Connections

```promql
# Active server connections (currently executing work)
sum(pgbouncer_pools_sv_active)

# Idle server connections (available for next request)
sum(pgbouncer_pools_sv_idle)

# Active per database
sum by (database) (pgbouncer_pools_sv_active)

# Idle per database
sum by (database) (pgbouncer_pools_sv_idle)
```

---

## Pool Utilization Percentage

```promql
# Overall pool utilization: active server connections as % of pool size
(
  sum(pgbouncer_pools_sv_active)
  /
  pgbouncer_config_default_pool_size
) * 100

# Per-database pool utilization
(
  sum by (database) (pgbouncer_pools_sv_active)
  /
  pgbouncer_config_default_pool_size
) * 100
```

---

## Total Client Connection Utilization

```promql
# Total connected clients (active + waiting) as % of max_client_conn
(
  sum(pgbouncer_pools_cl_active + pgbouncer_pools_cl_waiting)
  /
  pgbouncer_config_max_client_conn
) * 100

# How many client connections remain before hitting max_client_conn
pgbouncer_config_max_client_conn
-
sum(pgbouncer_pools_cl_active + pgbouncer_pools_cl_waiting)
```

---

## Active Clients

```promql
# Total active clients (all pools)
sum(pgbouncer_pools_cl_active)

# Active clients per database
sum by (database) (pgbouncer_pools_cl_active)

# Active clients per user
sum by (user) (pgbouncer_pools_cl_active)
```

---

## No Idle Servers Detection

```promql
# Pools with active clients but zero idle servers (saturated pools)
(sum by (database, user) (pgbouncer_pools_cl_active) > 0)
and
(sum by (database, user) (pgbouncer_pools_sv_idle) == 0)
```

---

## Exporter Health

```promql
# Is pgbouncer_exporter reachable?
pgbouncer_up

# Alert if absent or 0
pgbouncer_up == 0 or absent(pgbouncer_up)
```

---

## Time-Series Analysis

```promql
# How waiting client count changed over time
sum(pgbouncer_pools_cl_waiting)

# Peak waiting clients in the last 5 minutes
max_over_time(sum(pgbouncer_pools_cl_waiting)[5m:30s])

# Rate of change in active clients
rate(pgbouncer_pools_cl_active[5m])
```

---

## Dashboard Usage Tips

- Use `sum(pgbouncer_pools_cl_waiting)` in a stat panel — color it red when `> 0`.
- Use `sum(pgbouncer_pools_sv_idle)` in a stat panel — color it red when `== 0` (during active traffic).
- Use `sum by (database) (pgbouncer_pools_cl_active)` in a table panel for per-database breakdown.
- Pool utilization % gauge: set thresholds at 80% (yellow) and 95% (red).
