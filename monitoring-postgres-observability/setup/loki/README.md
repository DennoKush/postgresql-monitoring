# Loki Setup — Log Aggregation for Failed Database Connections

Loki runs on the **Observability Server**. It receives log streams pushed by Grafana Alloy on the PostgreSQL host and provides the LogQL query interface for Grafana dashboards and alert rules.

---

## Prerequisites

- Observability Server has Grafana already installed and running.
- Firewall allows inbound on port **3100** from the PostgreSQL host (for Alloy push).
- Grafana can reach Loki on port **3100** (localhost if co-located).

---

## 1. Install Loki

```bash
# Import the Grafana GPG key (skip if already done for Grafana)
sudo apt-get install -y apt-transport-https software-properties-common wget
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | \
  sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add the Grafana repository (skip if already added for Grafana)
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

# Install Loki
sudo apt-get update
sudo apt-get install -y loki

# Verify
loki --version
```

---

## 2. Create Data Directories

```bash
sudo mkdir -p /var/lib/loki/{chunks,rules,index,compactor}
sudo chown -R loki:loki /var/lib/loki
sudo chmod -R 750 /var/lib/loki
```

---

## 3. Deploy the Configuration

```bash
sudo cp /path/to/this/repo/setup/loki/loki-config.yml /etc/loki/loki-config.yml
sudo chown loki:loki /etc/loki/loki-config.yml
sudo chmod 640 /etc/loki/loki-config.yml
```

Review and adjust retention and resource limits in `loki-config.yml` for your environment.

---

## 4. Install the systemd Service

The `apt` package installs a systemd unit. Confirm the config path:

```bash
sudo systemctl cat loki
```

If the unit does not reference `/etc/loki/loki-config.yml`, override it:

```bash
sudo mkdir -p /etc/systemd/system/loki.service.d
sudo tee /etc/systemd/system/loki.service.d/override.conf <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/loki -config.file=/etc/loki/loki-config.yml
EOF
sudo systemctl daemon-reload
```

Enable and start:

```bash
sudo systemctl enable loki
sudo systemctl start loki
```

---

## 5. Configure the Grafana Loki Datasource

Deploy the provisioning file so Grafana discovers Loki automatically:

```bash
sudo cp /path/to/this/repo/setup/loki/loki-datasource.yml \
  /etc/grafana/provisioning/datasources/loki-datasource.yml
sudo chown grafana:grafana /etc/grafana/provisioning/datasources/loki-datasource.yml
sudo systemctl reload grafana-server
```

Or add it manually in Grafana UI:

1. **Connections → Add new datasource → Loki**
2. URL: `http://localhost:3100` (if Loki and Grafana are on the same host)
3. Click **Save & Test** — expected: `Data source connected and labels found`

---

## 6. Validate

See [`validation.md`](validation.md) for full validation steps.

Quick check:

```bash
sudo systemctl status loki
curl -s http://localhost:3100/ready
# Expected: ready
```

---

## Port Reference

| Port | Service | Direction | Notes |
|---|---|---|---|
| 3100 | Loki HTTP | Alloy → Loki (push) | Ingest endpoint |
| 3100 | Loki HTTP | Grafana → Loki (query) | Local if co-located |
| 9096 | Loki gRPC | Internal | Not exposed externally |

---

## Troubleshooting

### `loki: command not found` after apt install

```bash
which loki || ls /usr/bin/loki /usr/local/bin/loki 2>/dev/null
```

The binary may be at `/usr/local/bin/loki` for manual installs. Adjust the service `ExecStart` accordingly.

### Loki starts but Alloy cannot push

```bash
# From the PostgreSQL host
curl -s http://<OBSERVABILITY_SERVER_IP>:3100/ready
```

If this fails, check the firewall on the Observability Server:

```bash
sudo ufw status
# OR
sudo iptables -L -n | grep 3100
```

Allow inbound from the PostgreSQL host:

```bash
sudo ufw allow from <PG_HOST_IP> to any port 3100 proto tcp
```

### Loki ingests logs but labels are missing

Verify Alloy's `stage.labels` block in `config.alloy` includes `severity` and `error_type`. Then query the Loki label API:

```bash
curl -s http://localhost:3100/loki/api/v1/labels | python3 -m json.tool
```

### Disk usage grows too fast

Reduce the retention period in `loki-config.yml`:

```yaml
limits_config:
  retention_period: 72h   # reduce from 168h (7d) to 3d
```

Then restart Loki: `sudo systemctl restart loki`
