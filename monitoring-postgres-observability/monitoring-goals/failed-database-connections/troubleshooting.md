# Troubleshooting — Failed Database Connections

---

## Symptom: No FATAL log lines appear in the PostgreSQL log file

**Check 1 — Is `log_connections` enabled?**

```bash
sudo -u postgres psql -c "SHOW log_connections;"
```

Expected: `on`. If `off`, edit `postgresql.conf` and reload:

```bash
sudo -u postgres psql -c "SHOW config_file;"
# Edit the file, set log_connections = on
sudo systemctl reload postgresql
```

**Check 2 — Is `logging_collector` enabled?**

```bash
sudo -u postgres psql -c "SHOW logging_collector;"
```

Expected: `on`. If `off`, this requires a **restart** (not a reload):

```bash
sudo systemctl restart postgresql
```

**Check 3 — What is the log file path?**

```bash
sudo -u postgres psql -c "SELECT pg_current_logfile();"
sudo -u postgres psql -c "SHOW log_directory;"
sudo -u postgres psql -c "SHOW data_directory;"
```

The log is at `<data_directory>/<log_directory>/`. Check if the directory exists and is writable by `postgres`.

**Check 4 — Trigger a test event and watch the log in real time**

```bash
sudo tail -f <log_path>/postgresql-$(date +%Y-%m-%d).log &
psql -h 127.0.0.1 -U definitely_nonexistent_user -d postgres 2>&1 || true
```

---

## Symptom: Alloy is running but logs are not reaching Loki

**Check 1 — Alloy journal for errors**

```bash
sudo journalctl -u alloy -n 100 --no-pager | grep -iE "error|failed|refused|timeout"
```

Look for:
- `connection refused` → Loki is not running or port 3100 is blocked
- `permission denied` → Alloy cannot read the log file (see permissions fix below)
- `context deadline exceeded` → network timeout between hosts

**Check 2 — Confirm Loki is reachable from the PostgreSQL host**

```bash
curl -s http://<LOKI_URL>:3100/ready
```

Expected: `ready`. If not:
- Check `sudo systemctl status loki` on the Observability Server
- Check firewall: `sudo ufw status` and ensure port 3100 is allowed from the PostgreSQL host IP

**Check 3 — Confirm the log file path in config.alloy**

```bash
sudo cat /etc/alloy/config.alloy | grep __path__
```

Compare the path with the actual log file path:

```bash
sudo -u postgres psql -c "SELECT pg_current_logfile();"
```

If the paths differ (e.g., Alloy is watching `/var/log/postgresql/*.log` but logs go to `/var/lib/postgresql/18/main/log/*.log`), update `config.alloy` and restart Alloy:

```bash
sudo systemctl restart alloy
```

**Check 4 — Alloy cannot read the log directory**

```bash
sudo -u alloy ls /var/lib/postgresql/<version>/main/log/ 2>&1
```

If you see `Permission denied`:

```bash
sudo usermod -aG postgres alloy
sudo systemctl restart alloy
```

---

## Symptom: Loki receives logs but labels (severity, error_type) are missing

**Check 1 — Confirm the label pipeline ran**

Query the Loki labels API:

```bash
curl -s 'http://<LOKI_URL>:3100/loki/api/v1/labels' | python3 -m json.tool
```

If `severity` or `error_type` are absent, the Alloy pipeline stages did not fire.

**Check 2 — Verify the log line format matches the regex in config.alloy**

The Alloy `stage.regex` for severity uses:

```
\s(?P<severity>FATAL|ERROR|WARNING|PANIC):
```

Manually test this against a real log line:

```bash
sudo grep "FATAL" <log_path>/postgresql-$(date +%Y-%m-%d).log | head -1
```

The line must contain ` FATAL:` (with a space before FATAL and a colon after). If your `log_line_prefix` format differs, the regex may not match — adjust `stage.regex` in `config.alloy` accordingly.

**Check 3 — Check if the drop stage is discarding FATAL lines**

Temporarily comment out the `stage.drop` block in `config.alloy`, restart Alloy, and re-run the label query. If labels appear, the drop pattern is too broad and is also matching FATAL lines — refine the drop expression.

---

## Symptom: Grafana shows "No data" in the dashboard

**Check 1 — Correct datasource selected?**

Open the dashboard → **Dashboard settings → Variables** → confirm the datasource variable resolves to `Loki`.

**Check 2 — Is the time range appropriate?**

Set the Grafana time range to **Last 1 hour** and check if data appears. The default range may be narrower than the time of the last FATAL event.

**Check 3 — Run the query in Explore**

1. Open Grafana → **Explore** → select Loki
2. Run: `{job="postgres"}`
3. If data appears here but not in the dashboard, the panel query may have a different `job` label than what Alloy is using. Check `config.alloy`:
   ```
   job = "postgres"
   ```

---

## Symptom: Alert rules do not appear in Grafana after provisioning

**Check 1 — Provisioning YAML syntax**

```bash
sudo grafana-cli admin data-migration check
# or check Grafana logs:
sudo journalctl -u grafana-server -n 50 --no-pager | grep -i "alert\|provision\|error"
```

**Check 2 — File permissions**

```bash
ls -l /etc/grafana/provisioning/alerting/failed-db-connections-alerts.yml
```

The file must be readable by the `grafana` user:

```bash
sudo chown grafana:grafana /etc/grafana/provisioning/alerting/failed-db-connections-alerts.yml
sudo chmod 640 /etc/grafana/provisioning/alerting/failed-db-connections-alerts.yml
sudo systemctl reload grafana-server
```

**Check 3 — `datasourceUid` matches the provisioned Loki datasource**

The alert YAML uses `datasourceUid: loki`. Confirm the Loki datasource in Grafana has UID `loki`:

1. Open Grafana → **Connections → Data sources → Loki**
2. Check the URL — it ends in `…connections/datasources/edit/<uid>`
3. If the UID differs, update the YAML to match, or re-provision the datasource using `setup/loki/loki-datasource.yml` (which sets `uid: loki` explicitly).

---

## Symptom: `LokiPostgresLogIngestionStopped` fires even though PostgreSQL is running

**Check 1 — Is the log file being rotated to an unexpected path?**

Log rotation may have changed the filename (e.g., adding a timestamp suffix):

```bash
ls -lt /var/lib/postgresql/<version>/main/log/ | head -5
```

If the new file does not match the glob in `config.alloy` (`postgresql-*.log`), Alloy misses it. Adjust the glob pattern or the `log_filename` setting in `postgresql.conf`.

**Check 2 — Low-traffic environment with no log lines for 10m**

If PostgreSQL is genuinely quiet (no connections for 10 minutes), no log lines are produced and the alert fires legitimately. Options:

- Increase the alert window from `[10m]` to `[30m]`
- Add a PostgreSQL cron job or monitoring heartbeat that connects every few minutes to keep log output non-zero

**Check 3 — Silence during planned maintenance**

Use Grafana → **Alerting → Silences** to create a maintenance silence for the `LokiPostgresLogIngestionStopped` alert during planned downtime windows.
