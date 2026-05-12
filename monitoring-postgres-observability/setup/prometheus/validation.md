# Prometheus Validation

Run these checks after completing the installation in `README.md`.

---

## 1. Service Status

```bash
sudo systemctl status prometheus
```

Expected: `Active: active (running)`

---

## 2. Health Endpoint

```bash
curl -s http://localhost:9090/-/healthy
```

Expected: `Prometheus Server is Healthy.`

---

## 3. Configuration Validation

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Expected: `SUCCESS: 1 rule files found`  
(or similar — no FAILED lines)

---

## 4. Alert Rule Validation

```bash
promtool check rules /etc/prometheus/rules/postgres-connection-count.yml
promtool check rules /etc/prometheus/rules/pgbouncer-pool-exhaustion.yml
```

Expected: `SUCCESS: X rules found` for each file.

---

## 5. Scrape Targets — Both Must Be UP

```bash
curl -s 'http://localhost:9090/api/v1/targets?state=active' | python3 -m json.tool | grep -E '"job"|"health"|"lastError"'
```

Expected output for each target:
```
"job": "postgres_exporter",
"health": "up",
"lastError": "",
...
"job": "pgbouncer_exporter",
"health": "up",
"lastError": "",
```

Alternatively, open the Prometheus web UI:
```
http://<OBSERVABILITY_SERVER_IP>:9090/targets
```

Both `postgres_exporter` and `pgbouncer_exporter` jobs should show **State: UP** in green.

---

## 6. Verify Metrics in Prometheus

```bash
# Query total connection count
curl -s 'http://localhost:9090/api/v1/query?query=sum(pg_stat_activity_count)' | python3 -m json.tool

# Query PgBouncer waiting clients
curl -s 'http://localhost:9090/api/v1/query?query=sum(pgbouncer_pools_cl_waiting)' | python3 -m json.tool
```

Both should return `"status": "success"` with a numeric result.

---

## 7. Firewall — From PG17 Host to Observability Server

Confirm Prometheus can reach the exporters. From the Observability Server:

```bash
nc -zv <PG17_HOST_IP> 9187
nc -zv <PG17_HOST_IP> 9127
```

Expected: `Connection to <PG17_HOST_IP> 9187 port [tcp/*] succeeded!`

---

## 8. Logs

```bash
sudo journalctl -u prometheus -n 50 --no-pager
```

Look for any `level=error` lines related to scraping or configuration.

---

## Validation Checklist

- [ ] `systemctl status prometheus` shows `active (running)`
- [ ] `/-/healthy` returns healthy
- [ ] `promtool check config` passes
- [ ] Both rule files pass `promtool check rules`
- [ ] `postgres_exporter` target is UP in Prometheus
- [ ] `pgbouncer_exporter` target is UP in Prometheus
- [ ] `pg_stat_activity_count` query returns results
- [ ] `pgbouncer_pools_cl_waiting` query returns results
