# Grafana Alloy Setup — PostgreSQL Log Shipping

Grafana Alloy runs on the **PostgreSQL host**. It tails the PostgreSQL log files and ships matching log lines to Loki on the Observability Server.

Alloy replaces Promtail for new deployments and uses the River configuration language (`.alloy` files).

---

## Prerequisites

- PostgreSQL logging is configured and producing log files — see [`setup/postgres-logging/README.md`](../postgres-logging/README.md).
- Loki is running on the Observability Server — see [`setup/loki/README.md`](../loki/README.md).
- Firewall allows outbound from PostgreSQL host to Loki on port **3100**.

---

## 1. Install Grafana Alloy

```bash
# Import the Grafana GPG key
sudo apt-get install -y apt-transport-https software-properties-common wget
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | \
  sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add the Grafana repository
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

# Install Alloy
sudo apt-get update
sudo apt-get install -y alloy

# Verify
alloy --version
```

---

## 2. Create Configuration Directory

```bash
sudo mkdir -p /etc/alloy
sudo chown alloy:alloy /etc/alloy
```

---

## 3. Deploy the Configuration File

```bash
sudo cp /path/to/this/repo/setup/alloy/config.alloy /etc/alloy/config.alloy
sudo nano /etc/alloy/config.alloy
# Replace all <PLACEHOLDER> values
sudo chown alloy:alloy /etc/alloy/config.alloy
sudo chmod 640 /etc/alloy/config.alloy
```

Key placeholders to replace:

| Placeholder | Example value |
|---|---|
| `<POSTGRES_LOG_PATH>` | `/var/lib/postgresql/18/main/log` |
| `<PG_HOST_IP>` | `10.0.1.10` |
| `<LOKI_URL>` | `http://10.0.1.20:3100` |

---

## 4. Install and Enable the systemd Service

The `apt` package installs a systemd unit automatically. Confirm it references your config:

```bash
sudo systemctl cat alloy
```

If the `ExecStart` line does not point to `/etc/alloy/config.alloy`, override it:

```bash
sudo mkdir -p /etc/systemd/system/alloy.service.d
sudo tee /etc/systemd/system/alloy.service.d/override.conf <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/alloy run /etc/alloy/config.alloy --storage.path=/var/lib/alloy/data
EOF
sudo systemctl daemon-reload
```

Enable and start:

```bash
sudo systemctl enable alloy
sudo systemctl start alloy
```

---

## 5. Validate

See [`validation.md`](validation.md) for full validation steps.

Quick check:

```bash
sudo systemctl status alloy
sudo journalctl -u alloy -n 50 --no-pager
```

Look for lines such as:

```
msg="watching file" path=/var/lib/postgresql/.../log/postgresql-2025-05-17.log
```

---

## Troubleshooting

### Alloy cannot read the PostgreSQL log directory

**Symptom:** `permission denied` in Alloy journal logs.

**Fix:** Add the `alloy` user to the `postgres` group, or set the log directory permissions to allow group read:

```bash
# Option 1 — add alloy to postgres group
sudo usermod -aG postgres alloy
sudo systemctl restart alloy

# Option 2 — relax log directory permissions (read-only)
sudo chmod 750 /var/lib/postgresql/<version>/main/log
sudo chgrp alloy /var/lib/postgresql/<version>/main/log
```

Confirm Alloy can read the directory:

```bash
sudo -u alloy ls /var/lib/postgresql/<version>/main/log
```

### No logs appearing in Loki

1. Verify Loki is reachable from the PostgreSQL host:
   ```bash
   curl -s http://<LOKI_URL>:3100/ready
   # Expected: ready
   ```

2. Check Alloy logs for push errors:
   ```bash
   sudo journalctl -u alloy -n 100 --no-pager | grep -i error
   ```

3. Confirm the log file path in `config.alloy` matches the actual file path:
   ```bash
   sudo -u postgres psql -c "SELECT pg_current_logfile();"
   ```

### Alloy is shipping too many lines (noise)

Review the `stage.drop` block in `config.alloy`. Add additional drop patterns for noisy log lines that do not represent connection events:

```river
stage.drop {
  expression = "(?i)(autovacuum|checkpoint|LOG:  stats)"
}
```

### Log rotation breaks tailing

Alloy handles log rotation automatically by watching the file path glob (`*.log`). If rotation produces a new file, Alloy picks it up on the next poll cycle (default: 250ms). No manual intervention is needed.
