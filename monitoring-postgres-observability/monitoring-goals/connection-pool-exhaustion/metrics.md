# Metrics — Connection Pool Exhaustion

All metrics in this section are exposed by `pgbouncer_exporter` and scraped by Prometheus.

---

## Core Pool Metrics

### `pgbouncer_pools_cl_active`

Clients currently assigned a server connection (executing or in transaction).

- **Source:** `SHOW POOLS` — `cl_active` column
- **Type:** Gauge
- **Labels:** `database`, `user`, `pool_mode`

```
pgbouncer_pools_cl_active{database="mydb",pool_mode="transaction",user="appuser"} 8
```

### `pgbouncer_pools_cl_waiting`

Clients waiting for a server connection to become free.

- **Source:** `SHOW POOLS` — `cl_waiting` column
- **Type:** Gauge
- **Labels:** `database`, `user`, `pool_mode`

**This is the primary exhaustion signal. Any non-zero value means pool pressure.**

```
pgbouncer_pools_cl_waiting{database="mydb",pool_mode="transaction",user="appuser"} 3
```

### `pgbouncer_pools_sv_active`

Server connections currently assigned to a client.

- **Source:** `SHOW POOLS` — `sv_active` column
- **Type:** Gauge
- **Labels:** `database`, `user`, `pool_mode`

```
pgbouncer_pools_sv_active{database="mydb",pool_mode="transaction",user="appuser"} 15
```

### `pgbouncer_pools_sv_idle`

Server connections sitting idle, available for the next request.

- **Source:** `SHOW POOLS` — `sv_idle` column
- **Type:** Gauge
- **Labels:** `database`, `user`, `pool_mode`

**Zero idle servers with active clients = pool saturated.**

```
pgbouncer_pools_sv_idle{database="mydb",pool_mode="transaction",user="appuser"} 5
```

---

## Configuration Metrics

### `pgbouncer_config_max_client_conn`

The configured maximum total client connections to PgBouncer.

- **Source:** `SHOW CONFIG` — `max_client_conn`
- **Type:** Gauge

```
pgbouncer_config_max_client_conn 200
```

### `pgbouncer_config_default_pool_size`

The default number of server connections per database+user pool.

- **Source:** `SHOW CONFIG` — `default_pool_size`
- **Type:** Gauge

```
pgbouncer_config_default_pool_size 20
```

---

## Health Metric

### `pgbouncer_up`

Whether `pgbouncer_exporter` can connect to PgBouncer.

- **Type:** Gauge (0 or 1)

```
pgbouncer_up 1
```

---

## Additional Pool Metrics

| Metric | Description |
|---|---|
| `pgbouncer_pools_sv_used` | Server connections released but not yet cleaned up |
| `pgbouncer_pools_sv_tested` | Server connections being reset before reuse |
| `pgbouncer_pools_sv_login` | Server connections currently logging in |
| `pgbouncer_pools_maxwait` | Longest wait time (seconds) for a server connection |
| `pgbouncer_pools_maxwait_us` | `maxwait` in microseconds |
| `pgbouncer_databases_pool_size` | Pool size configured for a specific database |
| `pgbouncer_databases_max_connections` | Max server connections for a specific database |

---

## Metric Availability

These metrics appear only after PgBouncer has at least one active pool. If `pgbouncer_up 1` but no `pgbouncer_pools_*` metrics appear, no application has connected through PgBouncer yet. Make a test application connection and re-check.

```bash
# Check what pool metrics are available
curl -s http://localhost:9127/metrics | grep "^pgbouncer_pools"

# Check config metrics
curl -s http://localhost:9127/metrics | grep "^pgbouncer_config"
```
