# Grafana Setup

Grafana provides dashboards and alerting on top of Prometheus. All alerts in this stack are **Grafana-managed alerts** — Grafana evaluates rules against Prometheus and routes notifications.

---

## 1. Install Grafana

```bash
# Add Grafana APT repository
sudo apt install -y apt-transport-https software-properties-common
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key

echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install -y grafana

# Enable and start
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# Verify
sudo systemctl status grafana-server
```

---

## 2. Initial Login

Open a browser:
```
http://<OBSERVABILITY_SERVER_IP>:3000
```

Default credentials: `admin` / `admin`  
You will be prompted to change the password on first login.

---

## 3. Provision the Prometheus Datasource

Instead of using the UI, provision via file so it is reproducible:

```bash
sudo mkdir -p /etc/grafana/provisioning/datasources
sudo cp /path/to/this/repo/setup/grafana/datasource-prometheus.yml \
    /etc/grafana/provisioning/datasources/prometheus.yml
sudo systemctl restart grafana-server
```

Verify the datasource is connected:
1. Open Grafana → Connections → Data Sources → Prometheus
2. Click **Save & Test**
3. Expected: green banner — "Data source connected and labels found"

---

## 4. Import Dashboards

### Via the Grafana UI

1. Go to **Dashboards → Import**
2. Upload the JSON file from `setup/grafana/dashboards/`
3. Select the Prometheus datasource
4. Click **Import**

Import both:
- `postgres-connection-count.json`
- `pgbouncer-pool-exhaustion.json`

### Via provisioning (recommended for reproducibility)

```bash
sudo mkdir -p /etc/grafana/provisioning/dashboards
sudo mkdir -p /var/lib/grafana/dashboards

# Copy dashboard provider config
cat <<'EOF' | sudo tee /etc/grafana/provisioning/dashboards/postgres-observability.yml
apiVersion: 1
providers:
  - name: 'postgres-observability'
    orgId: 1
    folder: 'PostgreSQL Observability'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
EOF

# Copy dashboard JSON files
sudo cp /path/to/this/repo/setup/grafana/dashboards/*.json /var/lib/grafana/dashboards/
sudo chown -R grafana:grafana /var/lib/grafana/dashboards

sudo systemctl restart grafana-server
```

---

## 5. Configure Alert Rules

Alert rules are managed in Grafana UI under **Alerting → Alert Rules**. The rules are based on the Prometheus alert rule definitions in `setup/prometheus/rules/`.

To create a rule in Grafana:
1. Go to **Alerting → Alert Rules → New alert rule**
2. Select **Grafana managed alert**
3. Set the datasource to Prometheus
4. Enter the PromQL expression from the rule file
5. Configure thresholds, `for` duration, labels, and annotations
6. Assign to a folder and evaluation group

Repeat for all alert definitions in:
- [`setup/prometheus/rules/postgres-connection-count.yml`](../prometheus/rules/postgres-connection-count.yml)
- [`setup/prometheus/rules/pgbouncer-pool-exhaustion.yml`](../prometheus/rules/pgbouncer-pool-exhaustion.yml)

---

## 6. Configure Microsoft Teams Contact Point

See [`alerting/contact-point-teams.md`](alerting/contact-point-teams.md) for detailed steps.

---

## Troubleshooting

### Grafana datasource failing

**Symptom:** "Data source connected but no labels found" or connection error.

**Fix:**
1. Verify Prometheus is running: `systemctl status prometheus`
2. Verify datasource URL is `http://localhost:9090` (not an external IP, Grafana runs on the same server as Prometheus)
3. Check for typos in `datasource-prometheus.yml`

### Dashboards not appearing after provisioning

```bash
sudo journalctl -u grafana-server -n 30 --no-pager | grep -i "provision\|error"
```

Common causes:
- Wrong path in dashboard provider config
- JSON file permissions (must be readable by `grafana` user)
- Invalid JSON in dashboard file

### Logs

```bash
sudo journalctl -u grafana-server -n 50 --no-pager
# Or
sudo tail -f /var/log/grafana/grafana.log
```
