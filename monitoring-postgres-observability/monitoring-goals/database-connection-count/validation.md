# Validation — Database Connection Count

This end-to-end validation confirms the full monitoring path from PostgreSQL to Grafana is working for the connection count goal.

---

## Step 1 — Validate at the Source (PostgreSQL)

```bash
psql -h 127.0.0.1 -U monitoring_user -d postgres -c "
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;"
```

Note the counts. These should match what `postgres_exporter` exposes.

---

## Step 2 — Validate postgres_exporter

```bash
# Check exporter health
curl -s http://localhost:9187/metrics | grep pg_up
# Expected: pg_up 1

# Check connection count metrics
curl -s http://localhost:9187/metrics | grep pg_stat_activity_count | head -10
```

Pick one of the metric lines and verify its value matches the SQL query above.

---

## Step 3 — Validate Prometheus Is Scraping

From the Observability Server:

```bash
# Check target status
curl -s 'http://localhost:9090/api/v1/targets' | \
  python3 -c "import json,sys; data=json.load(sys.stdin); \
  [print(t['labels'].get('job',''), t['health']) for t in data['data']['activeTargets']]"
```

Expected: `postgres_exporter up`

```bash
# Query the metric in Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=sum(pg_stat_activity_count)' | \
  python3 -m json.tool | grep '"value"'
```

The numeric result should match the total from Step 1.

---

## Step 4 — Validate in Grafana

1. Open Grafana → **Explore** → select Prometheus datasource
2. Enter: `sum(pg_stat_activity_count)`
3. Click **Run query**
4. Confirm the value matches the SQL result from Step 1

---

## Step 5 — Validate Alert Rules

Validate the rule YAML files parse correctly:

```bash
promtool check rules /etc/prometheus/rules/postgres-connection-count.yml
```

Expected: `SUCCESS: 4 rules found`

---

## Step 6 — Test Alert Firing (Optional)

To test that alerts fire and reach Teams, temporarily lower a threshold in Grafana:

1. Go to **Alerting → Alert Rules → HighDatabaseConnectionUsage**
2. Edit the expression to use `> 0` instead of `> 70`
3. Wait for the `for` duration (5m or reduce to 1m for testing)
4. Verify the alert fires in Grafana and a notification appears in Teams
5. Restore the original threshold

---

## Validation Checklist

- [ ] SQL query on `pg_stat_activity` returns connection counts
- [ ] `postgres_exporter` shows `pg_up 1`
- [ ] `pg_stat_activity_count` metrics present in exporter output
- [ ] `prometheus_exporter` target shows `health: up` in Prometheus
- [ ] `sum(pg_stat_activity_count)` query returns correct value in Prometheus
- [ ] Dashboard panels show correct values in Grafana
- [ ] Alert rules pass `promtool check rules`
- [ ] Test alert fires and reaches Teams (if tested)
