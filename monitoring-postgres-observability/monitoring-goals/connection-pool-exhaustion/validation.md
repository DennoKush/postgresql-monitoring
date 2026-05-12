# Validation — Connection Pool Exhaustion

This end-to-end validation confirms the full monitoring path from PgBouncer to Grafana is working for the pool exhaustion goal.

---

## Step 1 — Validate at the Source (PgBouncer)

```bash
# Connect to PgBouncer admin database
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
```

Note the `cl_active`, `cl_waiting`, `sv_active`, and `sv_idle` values. These should match what `pgbouncer_exporter` exposes.

If no application pools appear (only the `pgbouncer` admin pool row), make a test connection:

```bash
psql -h 127.0.0.1 -p 6432 -U appuser -d mydb -c "SELECT 1;" &
```

Then rerun `SHOW POOLS`.

---

## Step 2 — Validate pgbouncer_exporter

```bash
# Check exporter health
curl -s http://localhost:9127/metrics | grep pgbouncer_up
# Expected: pgbouncer_up 1

# Check pool metrics
curl -s http://localhost:9127/metrics | grep "pgbouncer_pools_" | head -20
```

Verify that `pgbouncer_pools_cl_active` and `pgbouncer_pools_sv_idle` values match the `SHOW POOLS` output from Step 1.

---

## Step 3 — Validate Prometheus Is Scraping

From the Observability Server:

```bash
# Check target status
curl -s 'http://localhost:9090/api/v1/targets' | \
  python3 -c "import json,sys; data=json.load(sys.stdin); \
  [print(t['labels'].get('job',''), t['health']) for t in data['data']['activeTargets']]"
```

Expected: `pgbouncer_exporter up`

```bash
# Query waiting clients
curl -s 'http://localhost:9090/api/v1/query?query=sum(pgbouncer_pools_cl_waiting)' | \
  python3 -m json.tool | grep '"value"'

# Query active server connections
curl -s 'http://localhost:9090/api/v1/query?query=sum(pgbouncer_pools_sv_active)' | \
  python3 -m json.tool | grep '"value"'
```

---

## Step 4 — Validate in Grafana

1. Open Grafana → **Explore** → Prometheus datasource
2. Run: `sum(pgbouncer_pools_cl_waiting)` — should return 0 at idle baseline
3. Run: `sum(pgbouncer_pools_sv_idle)` — should return a positive number at idle baseline
4. Run: `pgbouncer_config_default_pool_size` — should match your `pgbouncer.ini` value

---

## Step 5 — Simulate Pool Pressure (Optional)

To verify alert firing works end-to-end, simulate waiting clients by reducing pool size below current demand.

**Warning: this affects all connections through PgBouncer. Only do this in a non-production environment.**

```bash
# Temporarily set pool size to 1 in pgbouncer.ini, reload
# Then create multiple concurrent connections
for i in $(seq 1 5); do
  psql -h 127.0.0.1 -p 6432 -U appuser -d mydb -c "SELECT pg_sleep(30);" &
done

# Observe SHOW POOLS
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
# Expected: cl_waiting > 0

# In Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=sum(pgbouncer_pools_cl_waiting)' | \
  python3 -m json.tool | grep '"value"'

# Restore after test
kill %1 %2 %3 %4 %5
# Restore default_pool_size in pgbouncer.ini and reload
sudo systemctl reload pgbouncer
```

---

## Step 6 — Validate Alert Rules

```bash
promtool check rules /etc/prometheus/rules/pgbouncer-pool-exhaustion.yml
```

Expected: `SUCCESS: X rules found`

---

## Validation Checklist

- [ ] `SHOW POOLS` returns application pool rows with expected values
- [ ] `pgbouncer_exporter` shows `pgbouncer_up 1`
- [ ] `pgbouncer_pools_cl_waiting` metric present (value 0 at baseline)
- [ ] `pgbouncer_pools_sv_idle` metric present (positive at baseline)
- [ ] `pgbouncer_config_default_pool_size` metric matches `pgbouncer.ini`
- [ ] `pgbouncer_exporter` target shows `health: up` in Prometheus
- [ ] PromQL queries return results in Prometheus
- [ ] Dashboard panels show values in Grafana
- [ ] Alert rules pass `promtool check rules`
