# Monitoring Goal: Failed Database Connections

## What Are Failed Database Connections?

A failed database connection is any connection attempt that PostgreSQL rejects before a session is established. These failures are terminal — the client receives a `FATAL` error and receives no database session.

PostgreSQL emits a `FATAL` log line for every rejected connection, including:

| Failure pattern | Cause |
|---|---|
| `FATAL: password authentication failed for user "<user>"` | Wrong password or SCRAM negotiation failure |
| `FATAL: role "<user>" does not exist` | The username has no matching PostgreSQL role |
| `FATAL: database "<database>" does not exist` | The requested database has been dropped or never existed |
| `FATAL: no pg_hba.conf entry for host "<ip>", user "<user>", database "<db>"` | The client IP/user/database combination is not in pg_hba.conf |
| `FATAL: connection rejected: pg_hba.conf rejects connection` | Explicit rejection rule in pg_hba.conf |
| SSL/certificate errors | TLS handshake failures on encrypted connections |

---

## Why It Matters

Failed connections are silent from the application's perspective — the error appears in the PostgreSQL log but not in any metric that `postgres_exporter` exposes. They cannot be detected from `pg_stat_activity` because the session never completes authentication.

| Risk | Consequence |
|---|---|
| Authentication failures | Broken application deployments; credential leaks or brute-force attempts |
| Unknown role | Application pointing at wrong username; missing role after migration |
| Unknown database | Wrong connection string; database dropped without updating app config |
| No pg_hba.conf entry | New host or IP range not yet granted access |
| SSL errors | TLS misconfiguration; certificate expiry approaching or expired |
| Repeated failures undetected | Security events go unnoticed; silent application failures |

A burst of failed connections is often the first indicator of a broken deployment, a configuration drift, or a security incident. Without log-based monitoring, these events are invisible to Prometheus-based alerting.

---

## Why Logs, Not Metrics?

`postgres_exporter` exposes metrics from PostgreSQL's internal statistics views (`pg_stat_activity`, `pg_settings`, etc.). These views only reflect **active sessions** — connections that have already authenticated. Rejected connections leave no trace in any system view.

The only authoritative record of a failed connection is the PostgreSQL log file.

This goal therefore uses a **log-based pipeline**:

```
PostgreSQL log file
  → Grafana Alloy (tail + label extraction)
  → Loki (log aggregation + LogQL)
  → Grafana (dashboard + alert rules)
  → Microsoft Teams (notifications)
```

---

## Signal Source

| Signal | Source |
|---|---|
| FATAL connection errors | PostgreSQL log file (requires `log_connections = on`) |
| Authentication failures | `|= "password authentication failed"` in LogQL |
| Unknown role errors | `|~ "role \".+\" does not exist"` in LogQL |
| Unknown database errors | `|~ "database \".+\" does not exist"` in LogQL |
| No pg_hba.conf entry errors | `|= "no pg_hba.conf entry"` in LogQL |
| SSL/certificate errors | `|~ "(?i)SSL\|certificate"` in LogQL |

All signals are sourced exclusively from PostgreSQL logs. `postgres_exporter` and `pgbouncer_exporter` are not used for this goal.

---

## Architecture

```
┌──────────────────────────────────────────┐     ┌──────────────────────────────────────┐
│              PostgreSQL Host              │     │         Observability Server          │
│                                          │     │                                      │
│  ┌─────────────────────────────────┐     │     │  ┌────────────────────────────────┐  │
│  │        PostgreSQL               │     │     │  │            Loki                │  │
│  │  log_connections = on           │     │     │  │  LogQL query engine            │  │
│  │  Writes FATAL lines to log      │     │     │  │  Log retention                 │  │
│  └──────────────┬──────────────────┘     │     │  └──────────────┬─────────────────┘  │
│                 │ log file               │     │                 │ queries            │
│  ┌──────────────▼──────────────────┐     │     │  ┌──────────────▼─────────────────┐  │
│  │        Grafana Alloy            │─────┼────►│  │           Grafana              │  │
│  │  Tails log files                │push │     │  │  Dashboard + Alert rules       │  │
│  │  Extracts severity / error_type │     │     │  └──────────────┬─────────────────┘  │
│  │  Ships to Loki                  │     │     │                 │ sends alerts       │
│  └─────────────────────────────────┘     │     │  ┌──────────────▼─────────────────┐  │
│                                          │     │  │      Microsoft Teams           │  │
└──────────────────────────────────────────┘     │  │      Alert channel             │  │
                                                 │  └────────────────────────────────┘  │
                                                 └──────────────────────────────────────┘
```

---

## Key Labels in Loki

Alloy extracts and promotes the following low-cardinality labels:

| Label | Values | Purpose |
|---|---|---|
| `job` | `postgres` | Identifies the log stream |
| `host` | PostgreSQL host IP | Identifies the source server |
| `severity` | `FATAL`, `ERROR`, `WARNING`, `PANIC` | Filters by log severity |
| `error_type` | `auth_failed`, `unknown_role`, `unknown_database`, `no_hba_entry`, `ssl_error` | Alert routing without raw user/IP data |

**Why not label by username or IP?** Raw usernames and IP addresses are high-cardinality values that would create unbounded label sets in Loki and inflate its index. Instead, error type is normalized into a small controlled vocabulary.

---

## Alert Summary

| Alert | Condition | For | Severity |
|---|---|---|---|
| FailedDatabaseConnectionsAboveBaseline | > 10 FATAL events in 5m | 5m | warning |
| FailedDatabaseAuthenticationSpike | > 5 auth failures in 2m | 2m | critical |
| RepeatedNoPgHbaEntryErrors | > 3 pg_hba.conf rejections in 5m | 5m | warning |
| RepeatedUnknownRoleConnectionAttempts | > 3 unknown-role errors in 5m | 5m | warning |
| RepeatedUnknownDatabaseConnectionAttempts | > 3 unknown-database errors in 5m | 5m | warning |
| LokiPostgresLogIngestionStopped | 0 log lines received in 10m | 10m | critical |

Adjust thresholds to reflect your environment's baseline.

---

## How to Visualize in Grafana

Import the dashboard from `setup/grafana/dashboards/failed-db-connections.json`.

Key panels:
1. **Failed Connections — Rate (per minute)** — time-series, broken out by error type
2. **Total Failed Connections** — stat panel, count over selected time range
3. **Authentication Failures** — stat with color threshold at 3 / 10
4. **Unknown Role Errors** — stat with color threshold at 1 / 5
5. **Unknown Database Errors** — stat with color threshold at 1 / 5
6. **No pg_hba.conf Entry Errors** — stat with color threshold at 1 / 5
7. **SSL / Certificate Errors** — stat with color threshold at 1 / 3
8. **Recent Failed Connection Log Lines** — live log panel filtered to `FATAL`
9. **Failed Connection Rate Over Time (5m buckets)** — time-series
10. **Top Failed Connection Messages** — table ranked by `error_type` count

---

## Setup Order

1. [`setup/postgres-logging/README.md`](../../setup/postgres-logging/README.md) — configure PostgreSQL to emit structured logs
2. [`setup/alloy/README.md`](../../setup/alloy/README.md) — install and configure Grafana Alloy on the PostgreSQL host
3. [`setup/loki/README.md`](../../setup/loki/README.md) — install Loki on the Observability Server
4. [`setup/loki/loki-datasource.yml`](../../setup/loki/loki-datasource.yml) — add Loki as a Grafana datasource
5. Import the dashboard from `setup/grafana/dashboards/failed-db-connections.json`
6. Deploy alert rules from `setup/grafana/alerting/failed-db-connections-alerts.yml`
7. Validate end-to-end: see [`validation.md`](validation.md)

---

## Troubleshooting

See [`troubleshooting.md`](troubleshooting.md) for detailed steps.

Quick checks:

```bash
# Is Alloy running and watching the log?
sudo systemctl status alloy
sudo journalctl -u alloy -n 30 --no-pager

# Is Loki up?
curl -s http://<LOKI_URL>:3100/ready

# Are log lines arriving in Loki?
curl -G 'http://<LOKI_URL>:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="postgres"}' \
  --data-urlencode 'limit=5' \
  --data-urlencode "start=$(date -d '5 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" | python3 -m json.tool | grep '"line"'
```
