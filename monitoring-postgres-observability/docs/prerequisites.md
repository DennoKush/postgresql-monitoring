# Prerequisites

## PostgreSQL 18 Host

| Requirement | Details |
|---|---|
| OS | Ubuntu Server 24.04 LTS |
| PostgreSQL | 17.x, already installed and running |
| CPU | 1+ vCPU (exporters are lightweight) |
| RAM | 512 MB minimum available for exporters |
| Network | Reachable from Observability Server on ports 9187 and 9127 |
| Internet access | Required during installation to download exporter binaries |
| Sudo access | Required for systemd service creation |

### PostgreSQL Configuration Requirements

Before deploying exporters, verify PostgreSQL is accepting connections and the config file is accessible:

```bash
# Verify PostgreSQL is running
sudo systemctl status postgresql

# Verify PostgreSQL version
psql --version
sudo -u postgres psql -c "SELECT version();"

# Confirm config file location (used in several steps)
sudo -u postgres psql -c "SHOW config_file;"
# Expected: /etc/postgresql/18/main/postgresql.conf

# Confirm hba file location
sudo -u postgres psql -c "SHOW hba_file;"
# Expected: /etc/postgresql/18/main/pg_hba.conf
```

### Required PostgreSQL User

A dedicated monitoring user must exist before deploying `postgres_exporter`. See [`setup/postgres-exporter/README.md`](../setup/postgres-exporter/README.md) for the exact SQL.

---

## Observability Server

| Requirement | Details |
|---|---|
| OS | Ubuntu Server 24.04 LTS |
| CPU | 2+ vCPU recommended |
| RAM | 2 GB minimum (Prometheus + Grafana) |
| Disk | 20 GB minimum for Prometheus time-series storage |
| Network | Can reach PG18 Host on ports 9187 and 9127 |
| Internet access | Required during installation |
| Sudo access | Required for systemd service creation |

---

## Network Connectivity Verification

Run from the **Observability Server** before deploying Prometheus:

```bash
# Replace <PG18_HOST_IP> with the actual IP address
curl -s http://<PG18_HOST_IP>:9187/metrics | head -20
curl -s http://<PG18_HOST_IP>:9127/metrics | head -20
```

If these return metrics, network connectivity is confirmed. If they time out or are refused, see [`docs/ports-and-networking.md`](ports-and-networking.md).

---

## Software Versions Used

| Component | Version | Source |
|---|---|---|
| postgres_exporter | 0.15.x | https://github.com/prometheus-community/postgres_exporter/releases |
| PgBouncer | 1.22.x | Ubuntu apt repository |
| pgbouncer_exporter | 0.7.x | https://github.com/prometheus-community/pgbouncer_exporter/releases |
| Prometheus | 2.51.x | https://github.com/prometheus/prometheus/releases |
| Grafana | 10.4.x | Grafana APT repository |

Check release pages for the latest stable versions before deploying. Installation steps in this guide use the versions above; adapt binary URLs accordingly.
