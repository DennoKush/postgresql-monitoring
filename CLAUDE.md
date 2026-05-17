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
│   ├── pgbouncer/
│   ├── pgbouncer-exporter/
│   ├── prometheus/
│   │   └── rules/      # postgres-connection-count.yml, pgbouncer-pool-exhaustion.yml
│   ├── grafana/
│   │   ├── dashboards/
│   │   └── alerting/
│   └── teams/
├── monitoring-goals/
│   ├── database-connection-count/
│   └── connection-pool-exhaustion/
└── runbooks/
```

## Architecture

The monitoring stack is split across two servers:

- **PostgreSQL host** (Ubuntu 24.04): runs `postgres_exporter` (port 9187) and `pgbouncer_exporter` (port 9127), both as systemd services. Applications connect through PgBouncer on port 6432, which pools connections to PostgreSQL on port 5432.
- **Observability server** (Ubuntu 24.04): runs Prometheus, Grafana, alert rules, and Microsoft Teams notification integration.

Prometheus scrapes both exporters by pulling metrics. Grafana reads Prometheus as its datasource and evaluates alert rules. Alert notifications are sent to Microsoft Teams via an incoming webhook.

## Key Constraints (from prompt.txt)

- **Do not** include failed database connection monitoring.
- **Do not** include pgaudit.
- **Do not** include Loki, Promtail, Grafana Alloy, mtail, or Vector.
- Secrets must always use `.env` files — never hardcoded.
- Prefer systemd-based services; Docker is allowed only where clearly justified.
- Use least-privilege PostgreSQL users for all exporters.
- Use IP/port placeholders: `<PG_HOST_IP>`, `<OBSERVABILITY_SERVER_IP>`, `<POSTGRES_EXPORTER_PORT>`, `<PGBOUNCER_EXPORTER_PORT>`.

## Monitoring Goals

Each goal has its own folder under `monitoring-goals/` with independent README, metrics, PromQL queries, Grafana dashboard description, alert rules, validation steps, and troubleshooting:

1. **Database Connection Count** — signals sourced from `postgres_exporter` / `pg_stat_activity`; uses `pg_stat_activity_count` PromQL metric.
2. **Connection Pool Exhaustion** — signals sourced from `pgbouncer_exporter`; `postgres_exporter` alone is insufficient for this goal.

## Alert Rules

| File | Alerts |
|---|---|
| `setup/prometheus/rules/postgres-connection-count.yml` | HighDatabaseConnectionUsage, DatabaseConnectionsNearMax, TooManyIdleConnections, SuddenConnectionSpike |
| `setup/prometheus/rules/pgbouncer-pool-exhaustion.yml` | PgBouncerClientsWaiting, PgBouncerPoolNearExhaustion, PgBouncerMaxClientConnNearLimit, PgBouncerNoIdleServers, PgBouncerExporterDown |

## Style Requirements

- Step-by-step numbered procedures throughout.
- All commands in fenced code blocks.
- Tables for ports, thresholds, and comparisons.
- Include validation commands after every major setup step.
- Include a troubleshooting section in each component README.
- Ubuntu 24.04 compatible paths (e.g., `/etc/postgresql/<version>/main/` for PostgreSQL config — substitute the actual installed major version).
