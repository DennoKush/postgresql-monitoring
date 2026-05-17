# Grafana Alloy Validation

Run these checks after completing the installation steps in `README.md`.

---

## 1. Service Status

```bash
sudo systemctl status alloy
```

Expected: `Active: active (running)`

---

## 2. Alloy is Watching the Log File

```bash
sudo journalctl -u alloy -n 100 --no-pager | grep -i "watching\|tagging\|path"
```

Expected output contains lines referencing the PostgreSQL log file path.

---

## 3. Trigger a Failed Connection and Confirm Shipping

In one terminal, tail Alloy's own journal:

```bash
sudo journalctl -u alloy -f
```

In a second terminal, generate a failed connection:

```bash
psql -h 127.0.0.1 -U bad_user_test -d postgres 2>&1 || true
```

Alloy should log a line indicating a new log entry was processed and shipped.

---

## 4. Confirm Logs Reach Loki

Query Loki directly from the Observability Server:

```bash
curl -G 'http://<LOKI_URL>:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={job="postgres"}' \
  --data-urlencode 'limit=5' \
  --data-urlencode "start=$(date -d '5 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000" | \
  python3 -m json.tool | grep '"line"' | head -5
```

Expected: one or more log lines containing `FATAL` from the test connection above.

---

## 5. Confirm Low-Cardinality Labels Are Present

```bash
curl -s 'http://<LOKI_URL>:3100/loki/api/v1/labels' | python3 -m json.tool
```

Expected: `job`, `host`, `severity`, `error_type` appear in the label list.

---

## 6. Alloy UI (Optional)

Alloy exposes a local UI on port 12345 (as configured in the service file):

```bash
curl -s http://127.0.0.1:12345/
```

Or open `http://<PG_HOST_IP>:12345` in a browser (only if the port is accessible and you have added a firewall rule). The UI shows the component graph and live log pipeline status.

---

## Validation Checklist

- [ ] `systemctl status alloy` shows `active (running)`
- [ ] Alloy journal shows the PostgreSQL log file being watched
- [ ] Failed test connection produces a `FATAL` line in the PostgreSQL log file
- [ ] Loki query returns log lines with `job="postgres"`
- [ ] Loki label list includes `severity` and `error_type`
