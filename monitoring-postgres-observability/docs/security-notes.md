# Security Notes

## PostgreSQL Monitoring User

The `postgres_exporter` user must have the minimum privileges needed to read system views. Never run it as a superuser.

```sql
-- Create a dedicated monitoring user
CREATE USER monitoring_user WITH PASSWORD '<strong_password>';

-- Grant pg_monitor role (PostgreSQL 10+) — provides read access to all monitoring views
GRANT pg_monitor TO monitoring_user;

-- Verify the grant
SELECT rolname, rolsuper, rolinherit
FROM pg_roles
WHERE rolname = 'monitoring_user';
```

The `pg_monitor` role provides read access to:
- `pg_stat_activity`
- `pg_stat_replication`
- `pg_stat_bgwriter`
- `pg_stat_database`
- `pg_settings`

It does **not** allow reading table data, modifying configuration, or executing DML.

---

## PgBouncer Admin User

The `pgbouncer_exporter` connects to PgBouncer's `pgbouncer` admin database. Create a dedicated user for this:

In `pgbouncer.ini`:
```ini
admin_users = pgb_exporter
stats_users = pgb_exporter
```

In `userlist.txt`:
```
"pgb_exporter" "<hashed_or_plain_password>"
```

This user can only run `SHOW` commands and cannot modify PgBouncer configuration.

---

## Secrets Management

All secrets must be stored in `.env` files, not hardcoded in configuration files or systemd unit files.

### File permissions

```bash
# .env files must not be world-readable
chmod 600 /etc/postgres_exporter/.env
chmod 600 /etc/pgbouncer-exporter/.env

# Service directories
chmod 750 /etc/postgres_exporter
chmod 750 /etc/pgbouncer-exporter

# Set ownership to the service user
chown postgres_exporter:postgres_exporter /etc/postgres_exporter/.env
chown pgb_exporter:pgb_exporter /etc/pgbouncer-exporter/.env
```

### systemd EnvironmentFile

Load secrets into the process environment without exposing them in `ps` output:

```ini
[Service]
EnvironmentFile=/etc/postgres_exporter/.env
ExecStart=/usr/local/bin/postgres_exporter
```

---

## Exporter Network Exposure

Exporter HTTP endpoints have no built-in authentication. They expose database internals (connection counts, query statistics, lock information). Treat them as sensitive.

Mitigations:
1. Firewall — allow port 9187 and 9127 only from the Observability Server IP (see [`docs/ports-and-networking.md`](ports-and-networking.md)).
2. If a reverse proxy is used, add basic auth or mutual TLS in front of exporter endpoints.
3. Do not expose exporter ports to the public internet.

---

## Grafana Teams Webhook URL

The Teams webhook URL is a secret that grants the ability to post messages to a Teams channel.

- Store it in Grafana's built-in encrypted secret store (configured via the Grafana UI under Contact Points — the value is not stored in plain text in the database after being set).
- Do not store the webhook URL in prometheus.yml, alert rule files, or any version-controlled file.
- Rotate the webhook URL if the Grafana instance is compromised.

---

## pg_hba.conf Considerations

The `postgres_exporter` connects from `localhost` (127.0.0.1) using password authentication:

```
# /etc/postgresql/18/main/pg_hba.conf
# Monitoring user — password auth from localhost only
host    postgres        monitoring_user     127.0.0.1/32    scram-sha-256
```

Do not use `trust` for the monitoring user, even on localhost. Use `scram-sha-256`.

After editing `pg_hba.conf`, reload PostgreSQL without a full restart:

```bash
sudo systemctl reload postgresql
# or
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

---

## systemd Service Users

Each exporter should run as a dedicated, non-login system user with minimal filesystem access:

```bash
# postgres_exporter
sudo useradd --system --no-create-home --shell /usr/sbin/nologin postgres_exporter

# pgbouncer_exporter
sudo useradd --system --no-create-home --shell /usr/sbin/nologin pgb_exporter
```

Prometheus and Grafana packages create their own service users automatically during installation.
