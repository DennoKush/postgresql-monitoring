# End-to-End Validation — Failed Database Connections

Run these checks after completing all setup steps. Each step validates one layer of the pipeline.

---

## 1. PostgreSQL Log File Is Active

```bash
sudo -u postgres psql -c "SELECT pg_current_logfile();"
```

Note the output path, e.g. `/var/lib/postgresql/18/main/log/postgresql-2025-05-17.log`.

Confirm the file exists and is being written to:

```bash
sudo ls -lh /var/lib/postgresql/<version>/main/log/
sudo tail -5 /var/lib/postgresql/<version>/main/log/postgresql-$(date +%Y-%m-%d).log
```

---

## 2. Failed Connection Produces a Log Line

Trigger a failed authentication in one terminal:

```bash
psql -h 127.0.0.1 -U validation_test_nonexistent -d postgres 2>&1 || true
```

Immediately check the log:

```bash
sudo grep "validation_test_nonexistent" \
  /var/lib/postgresql/<version>/main/log/postgresql-$(date +%Y-%m-%d).log
```

Expected:

```
2025-05-17 12:00:00 UTC [1234]: [1-1] user=validation_test_nonexistent,db=postgres,app=psql,client=127.0.0.1 FATAL:  role "validation_test_nonexistent" does not exist
```

If nothing appears, `log_connections = on` is not set or the logging_collector is off. See `setup/postgres-logging/README.md`.

---

## 3. Grafana Alloy Is Running

```bash
sudo systemctl status alloy
sudo journalctl -u alloy -n 30 --no-pager | tail -10
```

Expected: `Active: active (running)` and log lines showing the PostgreSQL log file being watched.

---

## 4. Alloy Ships the Log Line to Loki

Wait 30 seconds after step 2, then query Loki from the Observability Server:

```bash
curl -G 'http://<LOKI_URL>:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="postgres"} |= "FATAL"' \
  --data-urlencode 'limit=5' \
  --data-urlencode "start=$(date -d '5 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" | \
  python3 -m json.tool | grep '"line"'
```

Expected: at least one line containing `role "validation_test_nonexistent" does not exist`.

---

## 5. Loki Labels Are Correctly Extracted

```bash
curl -s 'http://<LOKI_URL>:3100/loki/api/v1/labels' | python3 -m json.tool
```

Expected: the label list includes `job`, `host`, `severity`, `error_type`.

Query for the `severity` label values:

```bash
curl -s 'http://<LOKI_URL>:3100/loki/api/v1/label/severity/values' | python3 -m json.tool
```

Expected: `["FATAL"]` (or `["FATAL","ERROR"]` if errors are also present).

---

## 6. Grafana Can Query Loki

1. Open Grafana → **Explore** → select Loki datasource
2. Run: `{job="postgres"} |= "FATAL"`
3. Expected: the test FATAL line from step 2 appears in the log results

---

## 7. Dashboard Loads Correctly

1. Open Grafana → **Dashboards** → **PostgreSQL — Failed Database Connections**
2. Verify the time range is set to **Last 15 minutes**
3. Expected:
   - **Total Failed Connections** stat shows ≥ 1 (from the test event)
   - **Unknown Role Errors** stat shows ≥ 1
   - **Recent Failed Connection Log Lines** panel shows the test FATAL line
   - The time-series panel shows a spike at the time of the test

---

## 8. Alert Rules Are Loaded

1. Open Grafana → **Alerting → Alert rules**
2. Confirm all six rules appear under **Failed Database Connections**:
   - FailedDatabaseConnectionsAboveBaseline
   - FailedDatabaseAuthenticationSpike
   - RepeatedNoPgHbaEntryErrors
   - RepeatedUnknownRoleConnectionAttempts
   - RepeatedUnknownDatabaseConnectionAttempts
   - LokiPostgresLogIngestionStopped
3. All rules should show state **Normal**

---

## 9. Trigger a Test Alert and Confirm Teams Notification

Generate enough events to cross the alert threshold. For `FailedDatabaseConnectionsAboveBaseline` (threshold: 10 in 5m), run 15 failed connections rapidly:

```bash
for i in $(seq 1 15); do
  psql -h 127.0.0.1 -U alert_test_user_$i -d postgres 2>/dev/null || true
done
```

Wait for the `for: 5m` period to elapse, then check:

1. Grafana → **Alerting → Alert rules** — `FailedDatabaseConnectionsAboveBaseline` should enter **Firing** state
2. Microsoft Teams alert channel — a notification should arrive within the next evaluation cycle

To stop the alert, wait for the events to age out of the 5-minute window or silence the alert in Grafana.

---

## Validation Checklist

- [ ] PostgreSQL log file exists and contains connection log lines
- [ ] Failed connection attempt produces a `FATAL` log line
- [ ] `systemctl status alloy` shows `active (running)`
- [ ] Loki receives log lines: `{job="postgres"}` query returns results
- [ ] Loki labels include `severity` and `error_type`
- [ ] Grafana Explore query `{job="postgres"} |= "FATAL"` returns results
- [ ] Dashboard panels populate with data
- [ ] All six alert rules appear in Grafana alerting
- [ ] Test alert fires and notification arrives in Microsoft Teams
