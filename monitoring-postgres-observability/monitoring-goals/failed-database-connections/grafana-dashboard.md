# Grafana Dashboard — Failed Database Connections

**Dashboard file:** `setup/grafana/dashboards/failed-db-connections.json`  
**Dashboard UID:** `failed-db-connections`  
**Datasource:** Loki (`uid: loki`)

---

## Importing the Dashboard

1. Open Grafana → **Dashboards → Import**
2. Upload `setup/grafana/dashboards/failed-db-connections.json`
3. Select the **Loki** datasource when prompted
4. Click **Import**

Or provision it automatically by placing the JSON in the Grafana dashboards provisioning directory:

```bash
sudo cp setup/grafana/dashboards/failed-db-connections.json \
  /etc/grafana/provisioning/dashboards/failed-db-connections.json
sudo systemctl reload grafana-server
```

If using provisioning, also ensure a dashboard provisioning config file exists at `/etc/grafana/provisioning/dashboards/dashboards.yml`:

```yaml
apiVersion: 1
providers:
  - name: default
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

---

## Panel Descriptions

### Panel 1 — Failed Connections Rate (per minute)

**Type:** Time-series  
**Position:** Full-width, top

Shows the rate of failed connection events over time, broken out by error type. Each series represents one failure category:

| Series | LogQL |
|---|---|
| All FATAL | `sum(rate({job="postgres"} \|= "FATAL" [$__interval]))` |
| Auth failures | `sum(rate({job="postgres"} \|= "password authentication failed" [$__interval]))` |
| Unknown role | `sum(rate({job="postgres"} \|~ \`role ".+" does not exist\` [$__interval]))` |
| Unknown database | `sum(rate({job="postgres"} \|~ \`database ".+" does not exist\` [$__interval]))` |
| No pg_hba entry | `sum(rate({job="postgres"} \|= "no pg_hba.conf entry" [$__interval]))` |

Use this panel to observe trends and spikes over time. A sudden spike in `All FATAL` that is not matched by any specific sub-series may indicate an unknown new error type.

---

### Panels 2–7 — Stat Panels (Count Over Time Range)

Six stat panels across the second row, one per error category. Each shows the count of events in the currently selected Grafana time range.

Color thresholds:

| Panel | Green | Yellow/Orange | Red |
|---|---|---|---|
| Total Failed Connections | 0–4 | 5–19 | ≥ 20 |
| Authentication Failures | 0–2 | 3–9 | ≥ 10 |
| Unknown Role Errors | 0 | 1–4 | ≥ 5 |
| Unknown Database Errors | 0 | 1–4 | ≥ 5 |
| No pg_hba Entry Errors | 0 | 1–4 | ≥ 5 |
| SSL / Certificate Errors | 0 | 1–2 | ≥ 3 |

Adjust thresholds in the dashboard JSON or the Grafana panel editor to match your environment's normal baseline.

---

### Panel 8 — Recent Failed Connection Log Lines

**Type:** Logs panel  
**Query:** `{job="postgres"} |= "FATAL"`

Shows the raw log lines for the most recent FATAL connection events. Useful for immediate incident diagnosis — you can see the exact username, database, client IP, and error message without needing to query Loki's API directly.

Sort order: descending (newest first).

---

### Panel 9 — Failed Connection Rate Over Time (5m buckets)

**Type:** Time-series  
**Query:** `sum(count_over_time({job="postgres"} |= "FATAL" [5m]))`

Shows the count of FATAL events per 5-minute window, not the per-second rate. Useful for alert context — you can see at a glance whether the alert threshold (e.g., > 10 in 5m) was crossed and for how long.

---

### Panel 10 — Top Failed Connection Messages

**Type:** Table  
**Query:** `sum by (error_type) (count_over_time({job="postgres", error_type!=""} [$__range]))`

Shows a ranked table of normalized `error_type` values and their event counts for the selected time range. Relies on the `error_type` label extracted by the Alloy pipeline.

Sorted descending by count. Useful for understanding which failure category is dominant during an incident.

---

## Recommended Time Ranges

| Scenario | Grafana time range |
|---|---|
| Active incident | Last 15 minutes |
| Post-incident review | Last 1 hour |
| Daily health check | Last 24 hours |
| Weekly trend | Last 7 days |

---

## Alert Integration

The dashboard's `failed-db-connections` UID is referenced in alert rule `annotations.runbook_url`. When an alert fires, the Teams notification includes a link that opens this dashboard directly.

Ensure `<GRAFANA_URL>` in the alert YAML is replaced with the actual Grafana base URL, e.g. `http://10.0.1.20:3000`.
