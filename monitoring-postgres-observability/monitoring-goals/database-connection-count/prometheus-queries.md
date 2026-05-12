# PromQL Queries — Database Connection Count

Use these queries in the Grafana Explore view, dashboard panels, or alert rules.

---

## Total Connection Count

```promql
# Total connections across all databases and users
sum(pg_stat_activity_count)

# Total connections — instant query for current value
sum(pg_stat_activity_count)
```

---

## Connections by State

```promql
# Break down by state (active, idle, idle in transaction, etc.)
sum by (state) (pg_stat_activity_count)

# Active connections only
sum(pg_stat_activity_count{state="active"})

# Idle connections only
sum(pg_stat_activity_count{state="idle"})

# Idle in transaction (potential lock holders — watch these)
sum(pg_stat_activity_count{state=~"idle in transaction.*"})
```

---

## Connections by Database

```promql
# Total connections per database, sorted descending
sort_desc(sum by (datname) (pg_stat_activity_count))

# Connections to a specific database
sum(pg_stat_activity_count{datname="mydb"})
```

---

## Connections by User

```promql
# Total connections per user, sorted descending
sort_desc(sum by (usename) (pg_stat_activity_count))

# Connections from a specific user
sum(pg_stat_activity_count{usename="appuser"})
```

---

## Connection Usage Percentage

```promql
# Percentage of max_connections in use (0–100)
(
  sum(pg_stat_activity_count)
  /
  pg_settings_max_connections
) * 100
```

This is the primary threshold metric for connection exhaustion alerts.

---

## Peak Connections (Time Window Analysis)

```promql
# Maximum connection count seen in the last 5 minutes (sampled every 30s)
max_over_time(sum(pg_stat_activity_count)[5m:30s])

# Maximum in the last 1 hour
max_over_time(sum(pg_stat_activity_count)[1h:1m])

# Average over the last 5 minutes
avg_over_time(sum(pg_stat_activity_count)[5m:30s])
```

---

## Connection Spike Detection

```promql
# Change in connection count over the last 5 minutes
sum(pg_stat_activity_count)
-
sum(pg_stat_activity_count) offset 5m

# Rate of change per second
rate(pg_stat_activity_count[5m])
```

---

## Idle vs Active Ratio

```promql
# Ratio of idle to active connections
sum(pg_stat_activity_count{state="idle"})
/
sum(pg_stat_activity_count{state="active"})
```

A ratio greater than 3–5x may indicate an oversized connection pool or connection leaks.

---

## max_connections Reference

```promql
# Current max_connections value
pg_settings_max_connections

# Remaining connection headroom
pg_settings_max_connections - sum(pg_stat_activity_count)
```

---

## Dashboard Usage Tips

- Use `sum(pg_stat_activity_count)` in stat panels with `Instant query` enabled for current-value display.
- Use `sum by (state) (pg_stat_activity_count)` in time-series panels to see trends over time.
- Use `sort_desc(sum by (datname) (pg_stat_activity_count))` with `Instant query` and `Table` visualization for a ranked breakdown.
- For the usage percentage gauge, set thresholds at 70 (yellow) and 90 (red).
