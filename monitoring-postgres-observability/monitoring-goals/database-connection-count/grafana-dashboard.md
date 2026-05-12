# Grafana Dashboard — Database Connection Count

Dashboard file: [`setup/grafana/dashboards/postgres-connection-count.json`](../../setup/grafana/dashboards/postgres-connection-count.json)

---

## Panel Descriptions

### Row 1 — Current State (Stat Panels)

| Panel | Query | Visualization | Thresholds |
|---|---|---|---|
| Total Active Connections | `sum(pg_stat_activity_count{state="active"})` | Stat | yellow ≥ 50, red ≥ 80 |
| Total Idle Connections | `sum(pg_stat_activity_count{state="idle"})` | Stat | yellow ≥ 30, red ≥ 60 |
| Connection Usage % | `(sum(pg_stat_activity_count) / pg_settings_max_connections) * 100` | Gauge | yellow ≥ 70%, red ≥ 90% |
| max_connections | `pg_settings_max_connections` | Stat | no threshold (reference) |

---

### Row 2 — Time-Series Trends

**Panel: Connection Count by State**
- Query: `sum by (state) (pg_stat_activity_count)`
- Visualization: Time series — stacked
- Purpose: Shows how connection state composition changes over time. A growing `idle` line without a corresponding `active` line indicates idle connection accumulation.

**Panel: Connection Usage % Over Time**
- Query: `(sum(pg_stat_activity_count) / pg_settings_max_connections) * 100`
- Visualization: Time series with threshold lines at 70% and 90%
- Purpose: Visual trend toward exhaustion; useful for capacity planning.

---

### Row 3 — Distribution Tables

**Panel: Connections by Database**
- Query: `sort_desc(sum by (datname) (pg_stat_activity_count))` (instant)
- Visualization: Table — sorted by count descending
- Purpose: Identify which database is consuming the most connections.

**Panel: Connections by User**
- Query: `sort_desc(sum by (usename) (pg_stat_activity_count))` (instant)
- Visualization: Table — sorted by count descending
- Purpose: Identify which user/application is consuming the most connections.

---

### Row 4 — Peak Analysis

**Panel: Peak vs Current Connection Count**
- Query A: `max_over_time(sum(pg_stat_activity_count)[5m:30s])` — Peak (5m)
- Query B: `sum(pg_stat_activity_count)` — Current
- Visualization: Time series — overlaid
- Purpose: Shows the gap between current load and recent peak. Helps detect transient spikes that may not be visible at the current moment.

---

## Drill-Down Queries (Explore)

When an alert fires or a spike is visible on the dashboard, use these in Grafana Explore:

```promql
# Which database spiked?
sort_desc(sum by (datname) (pg_stat_activity_count))

# Which user spiked?
sort_desc(sum by (usename) (pg_stat_activity_count))

# Are idle-in-transaction connections present?
sum(pg_stat_activity_count{state=~"idle in transaction.*"})

# What was the connection count 1 hour ago?
sum(pg_stat_activity_count) offset 1h
```

---

## Import Instructions

1. Grafana → **Dashboards → Import**
2. Click **Upload JSON file**
3. Select `setup/grafana/dashboards/postgres-connection-count.json`
4. Under **Options**, select the Prometheus datasource
5. Click **Import**

The dashboard will appear under **Dashboards** → folder **PostgreSQL Observability**.

---

## Suggested Alert Integration

Add an **Alert list** panel to the dashboard to show current alert states:

- Panel type: **Alert list**
- Show: Alerts in folder `PostgreSQL Observability`
- State filter: Firing, Pending

This gives at-a-glance visibility into active problems alongside the metrics panels.
