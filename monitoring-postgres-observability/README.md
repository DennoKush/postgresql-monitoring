# PostgreSQL Observability Stack — ProjectClearSight

A production-ready monitoring implementation for PostgreSQL 18 and PgBouncer, covering two key observability goals:

1. **Database Connection Count** — track active, idle, and waiting connections against `max_connections`.
2. **Connection Pool Exhaustion** — detect PgBouncer pool pressure before application errors occur.

---

## Architecture Overview

```
┌──────────────────────────────────────┐        ┌──────────────────────────────────────┐
│           PostgreSQL 18 Host          │        │         Observability Server          │
│                                      │        │                                      │
│  ┌─────────────────┐                 │        │  ┌────────────┐   ┌──────────────┐  │
│  │   PostgreSQL 18  │◄───────────────┼────────┼──│ Prometheus │   │   Grafana    │  │
│  └────────┬────────┘                 │        │  └─────┬──────┘   └──────┬───────┘  │
│           │                          │        │        │                  │          │
│  ┌────────▼────────┐                 │        │        │scrapes           │queries   │
│  │   PgBouncer      │                │        │        │                  │          │
│  └────────┬────────┘                 │        │        ▼                  ▼          │
│           │                          │        │  ┌─────────────────────────────┐     │
│  ┌────────▼────────┐  :9187          │        │  │     Prometheus TSDB          │     │
│  │postgres_exporter│◄────────────────┼────────┼──│  (metrics storage)          │     │
│  └─────────────────┘                 │        │  └─────────────────────────────┘     │
│                                      │        │                  │                   │
│  ┌─────────────────┐  :9127          │        │          alerts  │                   │
│  │pgbouncer_exporter◄────────────────┼────────┼──────────────────┘                  │
│  └─────────────────┘                 │        │                  │                   │
└──────────────────────────────────────┘        │          ┌───────▼──────┐            │
                                                │          │ Microsoft    │            │
                                                │          │   Teams      │            │
                                                │          └──────────────┘            │
                                                └──────────────────────────────────────┘
```

### Component Roles

| Component | Role | Host |
|---|---|---|
| PostgreSQL 18 | Database server | PG18 Host |
| PgBouncer | Connection pooler — sits between app and PostgreSQL | PG18 Host |
| postgres_exporter | Exposes PostgreSQL internal metrics (pg_stat_activity, etc.) on :9187 | PG18 Host |
| pgbouncer_exporter | Exposes PgBouncer pool metrics (SHOW POOLS, SHOW CLIENTS) on :9127 | PG18 Host |
| Prometheus | Scrapes exporters on a pull model; stores metrics as time-series | Observability Server |
| Grafana | Visualizes Prometheus metrics; evaluates alert rules; routes notifications | Observability Server |
| Microsoft Teams | Receives alert notifications from Grafana via incoming webhook | External |

**Why pull-based?** Prometheus pulls (scrapes) metrics from exporters rather than receiving pushes. This means the exporters are always the source of truth, and Prometheus controls the scrape interval. If an exporter goes silent, Prometheus detects the gap.

---

## Monitoring Goals

### Goal 1 — Database Connection Count

**Signal source:** `postgres_exporter` → `pg_stat_activity` view  
**Why it matters:** PostgreSQL has a hard limit (`max_connections`). When that limit is reached, new connection attempts fail with `FATAL: sorry, too many clients already`. Tracking connection usage percentage, idle connections, and per-database/user breakdown enables proactive intervention.

See: [`monitoring-goals/database-connection-count/README.md`](monitoring-goals/database-connection-count/README.md)

### Goal 2 — Connection Pool Exhaustion

**Signal source:** `pgbouncer_exporter` → PgBouncer `SHOW POOLS`, `SHOW CLIENTS`, `SHOW SERVERS`  
**Why it matters:** PgBouncer pool exhaustion causes application-level queueing and timeouts *before* PostgreSQL `max_connections` is hit. A pool can be exhausted while PostgreSQL itself has spare capacity. `postgres_exporter` alone cannot detect this condition — `pgbouncer_exporter` is required.

See: [`monitoring-goals/connection-pool-exhaustion/README.md`](monitoring-goals/connection-pool-exhaustion/README.md)

---

## Recommended Deployment Order

1. **PostgreSQL 18 Host**
   1. Create monitoring user in PostgreSQL (`pg_monitor` role)
   2. Install and configure PgBouncer
   3. Install and configure `postgres_exporter`
   4. Install and configure `pgbouncer_exporter`
   5. Open firewall ports 9187 and 9127 to the Observability Server

2. **Observability Server**
   1. Install Prometheus; configure `prometheus.yml` with both scrape targets
   2. Validate Prometheus targets show UP
   3. Install Grafana; configure Prometheus datasource
   4. Import dashboards
   5. Configure alert rules
   6. Configure Microsoft Teams contact point and notification policy
   7. Send a test alert

---

## Required Ports

| Port | Service | Direction | Notes |
|---|---|---|---|
| 5433 | PostgreSQL | PgBouncer → PostgreSQL | Local only |
| 6432 | PgBouncer | App → PgBouncer | Application-facing |
| 9187 | postgres_exporter | Observability Server → PG18 Host | Prometheus scrape |
| 9127 | pgbouncer_exporter | Observability Server → PG18 Host | Prometheus scrape |
| 9090 | Prometheus | Internal / admin | Web UI |
| 3000 | Grafana | Admin / dashboard users | Web UI |

---

## Assumptions

- PostgreSQL 18 is already installed and running on the PG18 host.
- Both servers run Ubuntu Server 24.04 LTS.
- No monitoring tools are pre-installed on either server.
- The Observability Server can reach ports 9187 and 9127 on the PG18 Host.
- `systemd` is used for all service management.
- Secrets are stored in `.env` files, not hardcoded in configuration.

---

## Security Considerations

- The `postgres_exporter` user has only `pg_monitor` role — no superuser, no write access.
- `postgres_exporter` and `pgbouncer_exporter` should listen on `0.0.0.0` only if Prometheus is on a separate host. Otherwise restrict to localhost.
- Firewall rules should allow port 9187 and 9127 only from the Observability Server IP.
- The Grafana Teams webhook URL is a secret — store it in Grafana's encrypted secret store, not in plain config files.
- PgBouncer admin access is restricted to `pgbouncer_exporter` user with a strong password.

See: [`docs/security-notes.md`](docs/security-notes.md)
