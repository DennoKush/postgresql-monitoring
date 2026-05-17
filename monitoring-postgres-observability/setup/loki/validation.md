# Loki Validation

Run these checks after completing the installation steps in `README.md`.

---

## 1. Service Status

```bash
sudo systemctl status loki
```

Expected: `Active: active (running)`

---

## 2. Loki Readiness

```bash
curl -s http://localhost:3100/ready
```

Expected: `ready`

If Loki is still initializing you may see `Ingester not ready`. Wait 10–30 seconds and retry.

---

## 3. Metrics Endpoint

```bash
curl -s http://localhost:3100/metrics | grep -E "^loki_build_info|^loki_ingester_chunks_stored_total"
```

Expected: one or more metric lines are returned.

---

## 4. Labels API (after first logs arrive)

Run this after Alloy has shipped at least one log line:

```bash
curl -s 'http://localhost:3100/loki/api/v1/labels' | python3 -m json.tool
```

Expected: `job`, `host`, `severity`, `error_type` are present in the label list.

---

## 5. Query Log Lines

```bash
curl -G 'http://localhost:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="postgres"}' \
  --data-urlencode 'limit=10' \
  --data-urlencode "start=$(date -d '15 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" | \
  python3 -m json.tool | grep '"line"' | head -10
```

Expected: returns log lines from the PostgreSQL log file.

---

## 6. Query a Specific Error Type

```bash
curl -G 'http://localhost:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="postgres"} |= "FATAL"' \
  --data-urlencode 'limit=5' \
  --data-urlencode "start=$(date -d '15 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" | \
  python3 -m json.tool | grep '"line"'
```

If no FATAL lines have occurred yet, generate one:

```bash
psql -h 127.0.0.1 -U nonexistent_user_test -d postgres 2>&1 || true
```

Wait ~30 seconds for Alloy to ship the line, then re-run the query.

---

## 7. Grafana Datasource Test

1. Open Grafana → **Connections → Data sources → Loki**
2. Click **Save & Test**
3. Expected: `Data source connected and labels found`

---

## 8. Reachable from PostgreSQL Host

On the PostgreSQL host, confirm Alloy can reach Loki:

```bash
curl -s http://<OBSERVABILITY_SERVER_IP>:3100/ready
# Expected: ready
```

---

## Validation Checklist

- [ ] `systemctl status loki` shows `active (running)`
- [ ] `curl http://localhost:3100/ready` returns `ready`
- [ ] Loki labels API returns `job`, `severity`, `error_type`
- [ ] LogQL query `{job="postgres"}` returns log lines
- [ ] LogQL query with `|= "FATAL"` returns a line after a test failed connection
- [ ] Grafana **Save & Test** on the Loki datasource passes
- [ ] Loki endpoint reachable from PostgreSQL host on port 3100
