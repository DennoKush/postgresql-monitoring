# Alert Rules — Failed Database Connections

All alert rules are defined in:  
[`setup/grafana/alerting/failed-db-connections-alerts.yml`](../../setup/grafana/alerting/failed-db-connections-alerts.yml)

These rules use Loki as the datasource and are evaluated by Grafana's unified alerting engine. Alerts are routed to Microsoft Teams via the contact point configured in `setup/teams/`.

---

## Alert Summary

| Alert Name | Condition | For | Severity |
|---|---|---|---|
| FailedDatabaseConnectionsAboveBaseline | > 10 FATAL events in 5m | 5m | warning |
| FailedDatabaseAuthenticationSpike | > 5 auth failures in 2m | 2m | critical |
| RepeatedNoPgHbaEntryErrors | > 3 pg_hba.conf rejections in 5m | 5m | warning |
| RepeatedUnknownRoleConnectionAttempts | > 3 unknown-role errors in 5m | 5m | warning |
| RepeatedUnknownDatabaseConnectionAttempts | > 3 unknown-database errors in 5m | 5m | warning |
| LokiPostgresLogIngestionStopped | < 1 log line received in 10m | 10m | critical |

---

## Alert Details

### FailedDatabaseConnectionsAboveBaseline

**Purpose:** Catch any unusual volume of failed connections, regardless of type, before more specific alerts fire.

**LogQL expression:**
```logql
sum(count_over_time({job="postgres"} |= "FATAL" [5m]))
```

**Threshold:** > 10

**Tuning guidance:**
- In quiet environments (e.g., internal tooling), lower the threshold to 3–5.
- In high-traffic environments, run a baseline query over 7 days and set the threshold at 2× the P95 value.
- `for: 5m` prevents transient events (e.g., a single deployment restart) from paging.

**Recommended action:**
1. Open the Failed Database Connections dashboard.
2. Check which stat panel is elevated (auth failures, unknown role, etc.).
3. Look at the logs panel for the raw error message and client IP.
4. Follow the relevant sub-alert runbook.

---

### FailedDatabaseAuthenticationSpike

**Purpose:** Detect credential failures that indicate a broken deployment or a brute-force/credential-stuffing attack. `critical` severity because the cause may be a security incident.

**LogQL expression:**
```logql
sum(count_over_time({job="postgres"} |= "password authentication failed" [2m]))
```

**Threshold:** > 5

**Tuning guidance:**
- `for: 2m` is intentionally short — authentication failures at scale are urgent.
- Lower the threshold to 2 for environments where auth failures should be extremely rare.
- Consider combining with a rate-of-change check if your environment has legitimate batch reconnects.

**Recommended action:**
1. Check the logs panel for the `user=` field in failing lines.
2. Determine if it is one username (application issue) or many (brute force).
3. If brute force: block the source IP at the firewall immediately; notify security.
4. If application: identify which service and deployment version changed; roll back or fix credentials.

---

### RepeatedNoPgHbaEntryErrors

**Purpose:** Detect when a host or application is being silently rejected by pg_hba.conf — a common misconfiguration after network changes or new deployments.

**LogQL expression:**
```logql
sum(count_over_time({job="postgres"} |= "no pg_hba.conf entry" [5m]))
```

**Threshold:** > 3

**Tuning guidance:**
- Any `pg_hba.conf` rejection should be unusual. A threshold of 1 is appropriate for very controlled environments.
- Raise to 5–10 during a migration window where new hosts are being connected in stages.

**Recommended action:**
1. Look at the log line for the client IP and database combination.
2. Determine if the host is expected (new app server, VPN change) or unexpected (scanner, external probe).
3. If expected: add an entry to `pg_hba.conf`, reload PostgreSQL.
4. If unexpected: block at the firewall and investigate.

---

### RepeatedUnknownRoleConnectionAttempts

**Purpose:** Catch applications connecting with a username that does not exist — usually a misconfiguration or a missing role after a migration.

**LogQL expression:**
```logql
sum(count_over_time({job="postgres"} |~ `role ".+" does not exist` [5m]))
```

**Threshold:** > 3

**Tuning guidance:**
- Should be near zero in a stable environment. Threshold of 1 is reasonable for low-traffic environments.
- Raise during planned role rename migrations.

**Recommended action:**
1. Identify the username from the log line.
2. Check whether the role was intentionally dropped (e.g., after a rename) or accidentally.
3. Either recreate the role or fix the application's connection string.

---

### RepeatedUnknownDatabaseConnectionAttempts

**Purpose:** Catch applications connecting to a database name that does not exist — wrong connection string or a database that was dropped.

**LogQL expression:**
```logql
sum(count_over_time({job="postgres"} |~ `database ".+" does not exist` [5m]))
```

**Threshold:** > 3

**Tuning guidance:**
- Should never occur in production if connection strings are correctly configured.
- Threshold of 1 is appropriate for most environments.

**Recommended action:**
1. Find the database name from the log line.
2. Determine if the database was dropped, renamed, or if the application has a wrong config.
3. Restore the database or update the application's connection string.

---

### LokiPostgresLogIngestionStopped

**Purpose:** Detect when the monitoring pipeline itself has failed. If no PostgreSQL log lines arrive in 10 minutes, failed connection alerting is blind — all other alerts in this group will not fire even if failures are occurring.

**LogQL expression:**
```logql
sum(count_over_time({job="postgres"} [10m]))
```

**Threshold:** < 1 (alert when count is zero)

**Tuning guidance:**
- PostgreSQL always produces at least some log output if active (connection lines, checkpoint logs, etc.).
- 10 minutes is conservative — reduce to 5 minutes for stricter coverage.
- If PostgreSQL is intentionally shut down for maintenance, silence this alert during the window.

**Recommended action:**
1. Check Grafana Alloy on the PostgreSQL host:
   ```bash
   sudo systemctl status alloy
   sudo journalctl -u alloy -n 50 --no-pager
   ```
2. Confirm the PostgreSQL log path in `config.alloy` still exists:
   ```bash
   ls -l <POSTGRES_LOG_PATH>/
   ```
3. Check Loki on the Observability Server:
   ```bash
   curl -s http://localhost:3100/ready
   sudo systemctl status loki
   ```
4. Check network connectivity between hosts on port 3100.

---

## Deploying Alert Rules

```bash
# Via Grafana provisioning
sudo cp setup/grafana/alerting/failed-db-connections-alerts.yml \
  /etc/grafana/provisioning/alerting/failed-db-connections-alerts.yml
sudo systemctl reload grafana-server
```

Verify the rules appear in Grafana:

1. Open Grafana → **Alerting → Alert rules**
2. Confirm all six rules appear under the **Failed Database Connections** group
3. Each rule should show state `Normal` if no events are firing

---

## Adjusting Alert Labels

These labels control notification routing:

```yaml
labels:
  severity: warning   # or critical
  team: dba
  component: postgres
  goal: failed-database-connections
```

Modify `team` and add labels to match your Grafana notification policy routing rules in `setup/teams/`.
