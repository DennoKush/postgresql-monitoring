# Monitoring Goal: Connection Pool Exhaustion

## What Is Connection Pool Exhaustion?

A connection pool (PgBouncer in this stack) maintains a fixed number of actual PostgreSQL connections (`default_pool_size`) and multiplexes many client connections across them. When all server connections in the pool are busy and a new client request arrives, the client must **wait** for a server connection to become free.

Pool exhaustion occurs when the pool cannot satisfy client demand:
- All server connections are active
- No idle server connections remain
- Clients queue up waiting

Unlike PostgreSQL `max_connections` exhaustion (which causes an immediate error), pool exhaustion causes **latency and timeouts** — clients are held in queue rather than rejected immediately. This makes pool exhaustion more insidious: applications slow down silently before they start failing.

---

## Why PgBouncer Matters

Without PgBouncer, every application thread holds a direct PostgreSQL connection. In a microservice or high-concurrency environment with hundreds of threads, this directly drives `max_connections` exhaustion.

PgBouncer absorbs the "thundering herd" by allowing many clients to share a small pool of server connections. A pool of 20 server connections can serve hundreds of concurrent clients — as long as each transaction completes quickly.

---

## How PgBouncer Fits in the Architecture

```
Application (hundreds of clients)
        │
        │  connect to port 6432
        ▼
    PgBouncer
        │  maintains pool of 20 server connections
        │  (default_pool_size = 20)
        ▼
  PostgreSQL 18
```

PgBouncer pool exhaustion can occur **even when PostgreSQL has spare capacity**. If the pool is set to 20 but 20 long-running transactions are holding all connections, new clients wait — regardless of what `max_connections` is set to.

---

## The Difference Between Two Exhaustion Types

| Aspect | PostgreSQL max_connections | PgBouncer Pool Exhaustion |
|---|---|---|
| What's the limit? | `max_connections` (e.g., 100) | `default_pool_size` per pool (e.g., 20) |
| What happens at limit? | New connections rejected immediately | Clients queue and wait |
| Visible symptom | `FATAL: sorry, too many clients already` | Slow queries, timeouts, `cl_waiting > 0` |
| Who detects it? | `postgres_exporter` (pg_stat_activity) | `pgbouncer_exporter` (SHOW POOLS) |
| Can it happen before PG exhaustion? | N/A | Yes — pool can be full while PG has spare capacity |

---

## Key PgBouncer Concepts

| Concept | Meaning |
|---|---|
| `cl_active` | Clients currently assigned a server connection |
| `cl_waiting` | Clients waiting for a server connection — **the primary exhaustion signal** |
| `sv_active` | Server (PostgreSQL) connections currently in use |
| `sv_idle` | Server connections available for the next request |
| `default_pool_size` | Max server connections per database+user pool |
| `reserve_pool_size` | Extra server connections reserved for bursts |
| `max_client_conn` | Hard ceiling on total client connections to PgBouncer |
| `pool_mode` | transaction, session, or statement |

---

## PgBouncer Manual Validation Commands

Connect to PgBouncer's admin database:

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer
```

Then run:

```sql
-- Pool state — the most important view for exhaustion detection
SHOW POOLS;

-- All connected clients and their states
SHOW CLIENTS;

-- All server (PostgreSQL) connections and their states
SHOW SERVERS;

-- Aggregate statistics
SHOW STATS;

-- PgBouncer configuration
SHOW CONFIG;

-- Databases and their pool settings
SHOW DATABASES;
```

**SHOW POOLS output columns to watch:**

| Column | Meaning | Exhaustion Signal |
|---|---|---|
| `cl_active` | Clients with a server | Normal operation |
| `cl_waiting` | Clients waiting | > 0 = pool pressure |
| `sv_active` | Active server connections | Near pool size = saturated |
| `sv_idle` | Free server connections | 0 with active clients = saturated |
| `sv_used` | Server connections released but not yet cleaned | — |
| `sv_tested` | Server connections being reset | — |
| `sv_login` | Server connections logging in | — |
| `maxwait` | Longest wait time in seconds | > 0 = clients waiting |

---

## Signal Source

| Signal | Source |
|---|---|
| Waiting clients (`cl_waiting`) | `pgbouncer_exporter` → `SHOW POOLS` |
| Active clients (`cl_active`) | `pgbouncer_exporter` → `SHOW POOLS` |
| Active server connections (`sv_active`) | `pgbouncer_exporter` → `SHOW POOLS` |
| Idle server connections (`sv_idle`) | `pgbouncer_exporter` → `SHOW POOLS` |
| Total client connections | `pgbouncer_exporter` → `SHOW CLIENTS` |
| `max_client_conn` | `pgbouncer_exporter` → `SHOW CONFIG` |
| `default_pool_size` | `pgbouncer_exporter` → `SHOW CONFIG` |

`postgres_exporter` alone **cannot provide these signals**. `pgbouncer_exporter` is required.

---

## Alert Thresholds

| Alert | Condition | Severity | For |
|---|---|---|---|
| PgBouncerClientsWaiting | `cl_waiting > 0` | warning | 1m |
| PgBouncerClientsWaitingCritical | `cl_waiting > 10` | critical | 2m |
| PgBouncerPoolNearExhaustion | `sv_active / pool_size > 80%` | warning | 5m |
| PgBouncerMaxClientConnNearLimit | `total_clients / max_client_conn > 85%` | critical | 5m |
| PgBouncerNoIdleServers | `cl_active > 0 AND sv_idle == 0` | warning | 3m |
| PgBouncerExporterDown | `pgbouncer_up == 0 or absent` | critical | 2m |

---

## Troubleshooting

See [`troubleshooting.md`](troubleshooting.md).

Quick checks:

```bash
# Is pgbouncer_exporter running?
sudo systemctl status pgbouncer-exporter
curl -s http://localhost:9127/metrics | grep pgbouncer_up

# Are there waiting clients right now?
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;" | grep -v "| 0 |"

# What does Prometheus see?
curl -s 'http://localhost:9090/api/v1/query?query=sum(pgbouncer_pools_cl_waiting)' | python3 -m json.tool
```
