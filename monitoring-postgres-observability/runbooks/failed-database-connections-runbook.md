# Runbook: Failed Database Connections

**Alerts covered:** FailedDatabaseConnectionsAboveBaseline, FailedDatabaseAuthenticationSpike, RepeatedNoPgHbaEntryErrors, RepeatedUnknownRoleConnectionAttempts, RepeatedUnknownDatabaseConnectionAttempts, LokiPostgresLogIngestionStopped

---

## Overview

Failed database connections are PostgreSQL connection attempts that are rejected before a session is established. The failure is logged as a `FATAL` line in the PostgreSQL log file and is invisible to Prometheus-based metrics.

This runbook covers six alert scenarios. Start at **Step 1** to confirm which alert fired, then jump to the relevant section.

---

## Severity Guide

| Alert | Urgency | Likely Impact |
|---|---|---|
| FailedDatabaseAuthenticationSpike | High — act immediately | Possible security incident or broken deployment |
| LokiPostgresLogIngestionStopped | High — monitoring blind spot | All failed-connection alerts are inactive |
| FailedDatabaseConnectionsAboveBaseline | Medium | Something changed — investigate source |
| RepeatedNoPgHbaEntryErrors | Medium | New host or misconfigured IP not yet granted access |
| RepeatedUnknownRoleConnectionAttempts | Medium | Role missing or wrong connection string |
| RepeatedUnknownDatabaseConnectionAttempts | Medium | Wrong database name or dropped database |

---

## Step 1 — Confirm the Alert and Identify the Type

Open the Grafana dashboard: **PostgreSQL — Failed Database Connections**

Set the time range to **Last 15 minutes**.

1. Which stat panel is elevated?
   - **Authentication Failures** → go to [Section A](#section-a--authentication-failures)
   - **No pg_hba.conf Entry Errors** → go to [Section B](#section-b--no-pg_hbaconf-entry-errors)
   - **Unknown Role Errors** → go to [Section C](#section-c--unknown-role-errors)
   - **Unknown Database Errors** → go to [Section D](#section-d--unknown-database-errors)
   - None — all FATAL elevated → go to [Section E](#section-e--elevated-fatals-with-no-specific-type)
   - Alert is `LokiPostgresLogIngestionStopped` → go to [Section F](#section-f--loki-ingestion-stopped)

2. Look at the **Recent Failed Connection Log Lines** panel for the raw log lines. Note:
   - The `user=` field — which username is failing?
   - The `client=` field — which IP is the source?
   - The `db=` field — which database is targeted?

---

## Section A — Authentication Failures

**Alert:** `FailedDatabaseAuthenticationSpike`

### A1 — Determine if this is one source or many

```bash
# SSH to the PostgreSQL host and search recent log lines
sudo grep "password authentication failed" \
  <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | \
  awk -F'client=' '{print $2}' | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

- If **one IP dominates** → likely a broken application deployment or misconfigured service. Go to A2.
- If **many different IPs** → possible brute-force or credential-stuffing attack. Go to A3.

### A2 — Single source (broken application)

1. Identify which application is using the failing username:
   ```bash
   sudo grep "password authentication failed" \
     <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | \
     grep "client=<source_ip>" | tail -5
   ```

2. Check if a recent deployment changed the credentials:
   - Review deployment logs for the application
   - Compare the environment variable / secret for `DB_PASSWORD` with the value in PostgreSQL

3. Fix the credentials (rotate the secret or update the connection string) and restart the application.

4. Verify failures stop:
   ```bash
   sudo tail -f <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | grep "authentication failed"
   ```

### A3 — Multiple sources (potential brute force)

1. **Immediately** block the source IPs at the firewall:
   ```bash
   # Block a specific IP from reaching PostgreSQL
   sudo ufw deny from <source_ip> to any port 5432
   sudo ufw deny from <source_ip> to any port 6432
   ```

2. Notify the security team with the source IPs and time window.

3. Review `pg_hba.conf` to ensure no overly permissive `host all all 0.0.0.0/0` entries exist:
   ```bash
   sudo grep -v "^#" /etc/postgresql/<version>/main/pg_hba.conf | grep "0.0.0.0"
   ```

4. Consider enabling `fail2ban` or a similar tool for repeated connection failures.

5. After the incident: rotate the passwords for any roles that were targeted.

---

## Section B — No pg_hba.conf Entry Errors

**Alert:** `RepeatedNoPgHbaEntryErrors`

### B1 — Identify the rejected connection details

```bash
sudo grep "no pg_hba.conf entry" \
  <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | tail -10
```

Note the client IP, user, and database from the log line:

```
FATAL:  no pg_hba.conf entry for host "192.168.5.10", user "appuser", database "myapp", SSL off
```

### B2 — Determine if the source is expected

- Is this IP a known application server or internal tool? → expected host, missing entry (go to B3)
- Is this IP unknown or external? → unexpected access attempt (go to Section A3)

### B3 — Add the host to pg_hba.conf

```bash
sudo nano /etc/postgresql/<version>/main/pg_hba.conf
```

Add the appropriate entry. Use the most restrictive form:

```
# Allow appuser to connect to myapp from the application server
host    myapp    appuser    192.168.5.10/32    scram-sha-256
```

Reload PostgreSQL:

```bash
sudo systemctl reload postgresql
```

Verify the reload took effect and the client can now connect:

```bash
psql -h 127.0.0.1 -U appuser -d myapp -c "SELECT 1;" 2>&1
```

---

## Section C — Unknown Role Errors

**Alert:** `RepeatedUnknownRoleConnectionAttempts`

### C1 — Identify the missing role

```bash
sudo grep "does not exist" \
  <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | \
  grep "role" | tail -10
```

Note the role name from: `FATAL:  role "olduser" does not exist`

### C2 — Determine the correct action

**If the role was accidentally dropped:**

```bash
sudo -u postgres psql -c "\du"  # list all roles
# If the role is missing, recreate it:
sudo -u postgres psql -c "CREATE ROLE olduser WITH LOGIN PASSWORD '<strong_password>';"
sudo -u postgres psql -c "GRANT <permissions> TO olduser;"
```

**If the application has the wrong username:**

1. Find which application is using the wrong username (look at `app=` and `client=` in the log line)
2. Update the application's connection string to use the correct role name
3. Restart the application

### C3 — Verify

Wait 1 minute and confirm failures stop appearing in the log:

```bash
sudo tail -f <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | grep "does not exist"
```

---

## Section D — Unknown Database Errors

**Alert:** `RepeatedUnknownDatabaseConnectionAttempts`

### D1 — Identify the missing database

```bash
sudo grep "does not exist" \
  <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | \
  grep "database" | tail -10
```

Note: `FATAL:  database "wrongdb" does not exist`

### D2 — List existing databases

```bash
sudo -u postgres psql -c "\l"
```

### D3 — Determine the correct action

**If the database name in the app is wrong (typo or wrong env):**

1. Find the application from `app=` and `client=` in the log line
2. Update the `DB_NAME` / connection string to the correct database name
3. Restart the application

**If the database was accidentally dropped:**

1. Confirm with the DBA team whether the database should exist
2. Restore from backup or recreate as appropriate
3. Update PgBouncer `pgbouncer.ini` if the database is listed there

---

## Section E — Elevated FATALs with No Specific Type

This means `FailedDatabaseConnectionsAboveBaseline` fired but none of the specific sub-alerts reached their threshold. The failure type may be:

- SSL/certificate related
- An explicit `reject` rule in pg_hba.conf
- A custom error type not covered by the sub-alerts

### E1 — Inspect the raw log lines

Open Grafana → Explore → Loki → run:

```logql
{job="postgres"} |= "FATAL"
```

Set the time range to the alert period and read the full log lines to identify the error message.

### E2 — Search on the PostgreSQL host

```bash
sudo grep "FATAL" \
  <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | \
  grep -v "password authentication\|does not exist\|no pg_hba" | tail -20
```

---

## Section F — Loki Ingestion Stopped

**Alert:** `LokiPostgresLogIngestionStopped`

This is a pipeline health alert — no PostgreSQL logs are arriving in Loki. While this alert fires, **all other Failed Database Connection alerts are blind**.

### F1 — Check Alloy on the PostgreSQL host

```bash
sudo systemctl status alloy
sudo journalctl -u alloy -n 50 --no-pager | grep -iE "error|fail|refused"
```

If Alloy is not running:

```bash
sudo systemctl start alloy
sudo systemctl status alloy
```

### F2 — Check Loki on the Observability Server

```bash
curl -s http://localhost:3100/ready
sudo systemctl status loki
sudo journalctl -u loki -n 50 --no-pager | grep -iE "error|panic"
```

If Loki is not running:

```bash
sudo systemctl start loki
```

### F3 — Check Network Connectivity

From the PostgreSQL host:

```bash
curl -s http://<OBSERVABILITY_SERVER_IP>:3100/ready
```

If this fails, check the firewall on the Observability Server:

```bash
sudo ufw status | grep 3100
# If missing:
sudo ufw allow from <PG_HOST_IP> to any port 3100 proto tcp
```

### F4 — Check Log File Path

Confirm the log path in `config.alloy` still matches the actual log path:

```bash
cat /etc/alloy/config.alloy | grep "__path__"
sudo -u postgres psql -c "SELECT pg_current_logfile();"
```

If the paths differ, update `config.alloy` and restart Alloy.

### F5 — Verify Recovery

After fixing the issue, confirm Loki is receiving logs:

```bash
curl -G 'http://<LOKI_URL>:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="postgres"}' \
  --data-urlencode 'limit=5' \
  --data-urlencode "start=$(date -d '5 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" | \
  python3 -m json.tool | grep '"line"'
```

The `LokiPostgresLogIngestionStopped` alert will resolve on the next evaluation cycle (within 1 minute of recovery).

---

## Step 2 — Verify Resolution

Across all sections, confirm the alert resolves:

1. Check Grafana → **Alerting → Alert rules** — the alert should return to **Normal**
2. Check the dashboard stat panels — event counts should drop back to baseline
3. Run a final check of the log file to confirm failures have stopped:
   ```bash
   sudo tail -20 <POSTGRES_LOG_PATH>/postgresql-$(date +%Y-%m-%d).log | grep FATAL
   ```

---

## Step 3 — Post-Incident Actions

1. **Document** the root cause: deployment, configuration drift, external scan, credential expiry, or pipeline failure.
2. **Prevent recurrence:**
   - Authentication failures: enforce secret rotation policies and deployment validation
   - pg_hba.conf rejections: add new hosts to pg_hba.conf as part of the server provisioning checklist
   - Unknown roles: add a pre-deployment check that validates the required roles exist
   - Unknown databases: add a connectivity smoke test to the deployment pipeline
3. **Review alert thresholds** — if the alert fired for a legitimate low-volume event, raise the threshold or add a silence for known maintenance windows.
4. **Test the pipeline** after any Alloy or Loki maintenance to confirm `LokiPostgresLogIngestionStopped` would detect a real outage.
