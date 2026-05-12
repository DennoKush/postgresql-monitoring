# Ports and Networking

## Port Reference

| Port | Protocol | Service | Host | Direction | Notes |
|---|---|---|---|---|---|
| 5433 | TCP | PostgreSQL | PG18 Host | Internal only | postgres_exporter → PostgreSQL; PgBouncer → PostgreSQL |
| 6432 | TCP | PgBouncer | PG18 Host | App → PG18 Host | Application connection pooler |
| 9187 | TCP | postgres_exporter | PG18 Host | Observability Server → PG18 Host | Prometheus scrape endpoint |
| 9127 | TCP | pgbouncer_exporter | PG18 Host | Observability Server → PG18 Host | Prometheus scrape endpoint |
| 9090 | TCP | Prometheus | Observability Server | Internal / admin | Web UI and query API |
| 3000 | TCP | Grafana | Observability Server | Users / admin | Dashboard and alerting UI |

---

## Firewall Configuration

### On the PG18 Host (UFW)

Open exporter ports only to the Observability Server:

```bash
# Replace <OBSERVABILITY_SERVER_IP> with the actual IP
sudo ufw allow from <OBSERVABILITY_SERVER_IP> to any port 9187 proto tcp comment "prometheus scrape postgres_exporter"
sudo ufw allow from <OBSERVABILITY_SERVER_IP> to any port 9127 proto tcp comment "prometheus scrape pgbouncer_exporter"

# Verify rules
sudo ufw status verbose
```

Do **not** expose ports 9187 or 9127 to 0.0.0.0/0 in production. These endpoints expose database internals without authentication.

### On the Observability Server (UFW)

Allow admin access to Prometheus and Grafana:

```bash
# Allow your admin workstation or CIDR to access Grafana
sudo ufw allow from <ADMIN_CIDR> to any port 3000 proto tcp comment "grafana dashboard"

# Prometheus does not need to be externally accessible
# Grafana queries it on localhost
```

---

## Connectivity Tests

### From the Observability Server

```bash
# Test TCP reachability
nc -zv <PG18_HOST_IP> 9187
nc -zv <PG18_HOST_IP> 9127

# Test metric endpoints
curl -s http://<PG18_HOST_IP>:9187/metrics | grep pg_up
curl -s http://<PG18_HOST_IP>:9127/metrics | grep pgbouncer_up

# Expected output for postgres_exporter
# pg_up 1

# Expected output for pgbouncer_exporter
# pgbouncer_up 1
```

### From the PG18 Host (self-check)

```bash
# Verify postgres_exporter is listening
ss -tlnp | grep 9187

# Verify pgbouncer_exporter is listening
ss -tlnp | grep 9127

# Test locally
curl -s http://localhost:9187/metrics | head -5
curl -s http://localhost:9127/metrics | head -5
```

---

## Common Networking Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Connection refused` on port 9187 | postgres_exporter not running | `sudo systemctl start postgres_exporter` |
| `Connection timed out` on port 9187 | Firewall blocking | Check UFW rules on PG18 Host |
| Prometheus target shows DOWN | Exporter listening on 127.0.0.1 only | Change `--web.listen-address` to `0.0.0.0:9187` in service file |
| `Connection refused` on port 9127 | pgbouncer_exporter not running | `sudo systemctl start pgbouncer-exporter` |
| Grafana datasource error | Prometheus not reachable on 9090 | Verify Prometheus is running and datasource URL is `http://localhost:9090` |

---

## Exporter Listen Address

By default, configure exporters to listen on all interfaces when Prometheus is on a separate host:

```bash
# postgres_exporter
--web.listen-address=0.0.0.0:9187

# pgbouncer_exporter
--web.listen-address=0.0.0.0:9127
```

If Prometheus and the exporters are on the same host, restrict to `127.0.0.1`.
