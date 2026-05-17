# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

ProjectClearSight is a PostgreSQL monitoring observability implementation guide. The goal is to generate a complete, production-ready documentation and configuration artifact under the directory `monitoring-postgres-observability/` as described in `prompt.txt`.

There is no application code. The deliverables are Markdown documentation files, YAML/INI configuration files, systemd unit files, and Grafana JSON dashboards.

## Target Output Structure

```
monitoring-postgres-observability/
├── README.md
├── docs/               # architecture.md, prerequisites.md, ports-and-networking.md, security-notes.md
├── setup/
│   ├── postgres-exporter/
│   ├── postgres-logging/           # PostgreSQL log config for Failed DB Connections goal
│   ├── alloy/                      # Grafana Alloy — log shipping agent on PostgreSQL host
│   ├── pgbouncer/
│   ├── pgbouncer-exporter/
│   ├── prometheus/
│   │   └── rules/      # postgres-connection-count.yml, pgbouncer-pool-exhaustion.yml
│   ├── loki/                       # Loki — log aggregation on Observability Server
│   ├── grafana/
│   │   ├── dashboards/
│   │   └── alerting/
│   └── teams/
├── monitoring-goals/
│   ├── database-connection-count/
│   ├── connection-pool-exhaustion/
│   └── failed-database-connections/   # NEW — log-based goal using Alloy + Loki
└── runbooks/
```

## Architecture

The monitoring stack is split across two servers:

- **PostgreSQL host** (Ubuntu 24.04): runs `postgres_exporter` (port 9187), `pgbouncer_exporter` (port 9127), and `grafana-alloy` (tails PostgreSQL logs, pushes to Loki on port 3100). All run as systemd services. Applications connect through PgBouncer on port 6432, which pools connections to PostgreSQL on port 5432.
- **Observability server** (Ubuntu 24.04): runs Prometheus, Loki, Grafana, alert rules, and Microsoft Teams notification integration.

Prometheus scrapes both exporters by pulling metrics. Grafana Alloy pushes PostgreSQL logs to Loki. Grafana reads both Prometheus and Loki as datasources and evaluates alert rules. Alert notifications are sent to Microsoft Teams via an incoming webhook.

## Key Constraints

### Goals 1 & 2 (Database Connection Count, Connection Pool Exhaustion)
- **Do not** include Loki, Promtail, Grafana Alloy, mtail, or Vector.
- Signal source is `postgres_exporter` and `pgbouncer_exporter` only.

### Goal 3 (Failed Database Connections)
- **Do not** include pgaudit.
- **Do not** use `postgres_exporter` or `pgbouncer_exporter` as the primary signal source.
- Signal source is PostgreSQL log files → Grafana Alloy → Loki.
- Loki and Grafana Alloy are intentionally used for this goal only.

### All Goals
- Secrets must always use `.env` files — never hardcoded.
- Prefer systemd-based services; Docker is allowed only where clearly justified.
- Use least-privilege PostgreSQL users for all exporters.
- Use IP/port placeholders: `<PG_HOST_IP>`, `<OBSERVABILITY_SERVER_IP>`, `<POSTGRES_EXPORTER_PORT>`, `<PGBOUNCER_EXPORTER_PORT>`, `<LOKI_URL>`.
- Do not promote high-cardinality values (usernames, raw IPs) as Loki labels.

## Monitoring Goals

Each goal has its own folder under `monitoring-goals/` with independent README, signal source docs, query reference, Grafana dashboard description, alert rules, validation steps, and troubleshooting:

1. **Database Connection Count** — signals sourced from `postgres_exporter` / `pg_stat_activity`; uses `pg_stat_activity_count` PromQL metric.
2. **Connection Pool Exhaustion** — signals sourced from `pgbouncer_exporter`; `postgres_exporter` alone is insufficient for this goal.
3. **Failed Database Connections** — signals sourced from PostgreSQL log files via Grafana Alloy → Loki; uses LogQL queries; `postgres_exporter` cannot detect these events.

## Alert Rules

| File | Engine | Alerts |
|---|---|---|
| `setup/prometheus/rules/postgres-connection-count.yml` | Prometheus/Grafana | HighDatabaseConnectionUsage, DatabaseConnectionsNearMax, TooManyIdleConnections, SuddenConnectionSpike |
| `setup/prometheus/rules/pgbouncer-pool-exhaustion.yml` | Prometheus/Grafana | PgBouncerClientsWaiting, PgBouncerPoolNearExhaustion, PgBouncerMaxClientConnNearLimit, PgBouncerNoIdleServers, PgBouncerExporterDown |
| `setup/grafana/alerting/failed-db-connections-alerts.yml` | Grafana/Loki | FailedDatabaseConnectionsAboveBaseline, FailedDatabaseAuthenticationSpike, RepeatedNoPgHbaEntryErrors, RepeatedUnknownRoleConnectionAttempts, RepeatedUnknownDatabaseConnectionAttempts, LokiPostgresLogIngestionStopped |

## Style Requirements

- Step-by-step numbered procedures throughout.
- All commands in fenced code blocks.
- Tables for ports, thresholds, and comparisons.
- Include validation commands after every major setup step.
- Include a troubleshooting section in each component README.
- Ubuntu 24.04 compatible paths (e.g., `/etc/postgresql/<version>/main/` for PostgreSQL config — substitute the actual installed major version).
