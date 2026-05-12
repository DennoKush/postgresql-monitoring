# Prometheus Setup

Prometheus scrapes metrics from both exporters on the PG18 Host and stores them as time-series data for Grafana to query.

---

## 1. Install Prometheus

```bash
# Download latest stable release
wget https://github.com/prometheus/prometheus/releases/download/v2.51.0/prometheus-2.51.0.linux-amd64.tar.gz

# Verify checksum
sha256sum prometheus-2.51.0.linux-amd64.tar.gz

# Extract
tar -xzf prometheus-2.51.0.linux-amd64.tar.gz
cd prometheus-2.51.0.linux-amd64

# Install binaries
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/
sudo chmod +x /usr/local/bin/prometheus /usr/local/bin/promtool

# Install web UI assets
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r consoles/ console_libraries/ /etc/prometheus/

# Create service user
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Verify
prometheus --version
```

---

## 2. Configure prometheus.yml

```bash
sudo cp /path/to/this/repo/setup/prometheus/prometheus.yml /etc/prometheus/prometheus.yml
sudo nano /etc/prometheus/prometheus.yml
# Replace all placeholder values with actual IPs
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

---

## 3. Install Alert Rules

```bash
sudo mkdir -p /etc/prometheus/rules
sudo cp /path/to/this/repo/setup/prometheus/rules/*.yml /etc/prometheus/rules/
sudo chown -R prometheus:prometheus /etc/prometheus/rules/

# Validate rule files
promtool check rules /etc/prometheus/rules/postgres-connection-count.yml
promtool check rules /etc/prometheus/rules/pgbouncer-pool-exhaustion.yml
```

---

## 4. Install the systemd Service

Create `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus Monitoring Server
Documentation=https://prometheus.io/docs
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/data \
  --storage.tsdb.retention.time=30d \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.console.templates=/etc/prometheus/consoles \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

---

## 5. Validate

See [`validation.md`](validation.md).

Quick check:

```bash
sudo systemctl status prometheus

# Access web UI (from a browser or with curl)
curl -s http://localhost:9090/-/healthy
# Expected: Prometheus Server is Healthy.

# Check targets
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep '"health"'
```

---

## Reloading Configuration

After changing `prometheus.yml` or rule files, reload without restart (requires `--web.enable-lifecycle`):

```bash
curl -X POST http://localhost:9090/-/reload
```

Or restart the service:

```bash
sudo systemctl restart prometheus
```

---

## Troubleshooting

### Prometheus target shows DOWN

**Step 1 — Check the target error message:**
```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -A5 '"health": "down"'
```

**Step 2 — Verify network connectivity from Observability Server:**
```bash
curl -s http://<PG18_HOST_IP>:9187/metrics | head -5
curl -s http://<PG18_HOST_IP>:9127/metrics | head -5
```

**Step 3 — Check firewall on PG18 Host:**
```bash
sudo ufw status verbose | grep -E "9187|9127"
```

**Step 4 — Verify exporter listen address:**
On the PG18 Host:
```bash
ss -tlnp | grep -E "9187|9127"
# Should show 0.0.0.0 not 127.0.0.1 if Prometheus is remote
```

### Configuration validation

```bash
promtool check config /etc/prometheus/prometheus.yml
```

### Logs

```bash
sudo journalctl -u prometheus -n 50 --no-pager
```
