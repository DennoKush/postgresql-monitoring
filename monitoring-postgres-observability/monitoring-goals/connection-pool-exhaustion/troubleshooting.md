# Troubleshooting — Connection Pool Exhaustion

---

## pgbouncer_exporter Not Starting

**Symptom:** `systemctl status pgbouncer-exporter` shows `failed`.

**Diagnosis:**
```bash
sudo journalctl -u pgbouncer-exporter -n 50 --no-pager
```

**Common causes:**

| Log message | Fix |
|---|---|
| `connection refused on port 6432` | PgBouncer not running — `sudo systemctl start pgbouncer` |
| `password authentication failed` | Wrong password in `.env` |
| `no such file` for binary | Re-run installation step |
| `permission denied` on `.env` | `sudo chown pgb_exporter /etc/pgbouncer-exporter/.env` |

---

## PgBouncer Admin Database Not Reachable

**Symptom:** `pgbouncer_up 0`, or `psql: FATAL: no such user: pgb_exporter`

**Diagnosis:**
```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW VERSION;"
```

**Fix:**
1. Confirm `admin_users = pgb_exporter` or `stats_users = pgb_exporter` in `/etc/pgbouncer/pgbouncer.ini`.
2. Confirm `"pgb_exporter" "<password>"` exists in `/etc/pgbouncer/userlist.txt`.
3. Reload PgBouncer:
   ```bash
   sudo systemctl reload pgbouncer
   ```
4. Verify the password in `userlist.txt` matches the password in `/etc/pgbouncer-exporter/.env`.

---

## pgbouncer_exporter Metric Names Not Matching

**Symptom:** Metrics endpoint returns data but PromQL queries in dashboard/alerts return no data.

**Diagnosis:**
```bash
curl -s http://localhost:9127/metrics | grep "^pgbouncer" | head -20
```

Compare the metric name prefix with what the alert rules expect.

**Fix:**

| Version | Metric prefix |
|---|---|
| pgbouncer_exporter 0.7.x | `pgbouncer_pools_cl_active` |
| Some older forks | `pgbouncer2_pools_cl_active` |

If your metric names differ from the alert rules, either:
- Upgrade to the 0.7.x series, or
- Update all PromQL expressions in alert rules and dashboard panels to match the actual prefix.

---

## PgBouncer Pool Metrics Missing

**Symptom:** `pgbouncer_up 1` but no `pgbouncer_pools_*` metrics.

**Cause:** No application pool exists yet — `SHOW POOLS` only returns pools that have had at least one connection.

**Fix:** Make a test application connection through PgBouncer:
```bash
psql -h 127.0.0.1 -p 6432 -U appuser -d mydb -c "SELECT 1;"
```

Then re-check:
```bash
curl -s http://localhost:9127/metrics | grep pgbouncer_pools_
```

---

## PgBouncer SHOW Commands Failing

**Symptom:** `ERROR: permission denied` when running `SHOW POOLS` as `pgb_exporter`.

**Fix:**
1. The user must be listed in `stats_users` (for SHOW commands) or `admin_users` (for all admin commands) in `pgbouncer.ini`.
2. `stats_users` is sufficient for the exporter — it allows all `SHOW` commands.
3. After editing `pgbouncer.ini`, reload: `sudo systemctl reload pgbouncer`.

---

## PgBouncer Pool Exhaustion Occurring (Actual Problem)

If `cl_waiting > 0` is a real condition in your environment:

### Step 1 — Identify stuck transactions

```bash
psql -h 127.0.0.1 -U monitoring_user -d postgres -c "
SELECT pid, usename, datname, state, 
       now() - query_start AS query_duration,
       now() - state_change AS state_duration,
       query
FROM pg_stat_activity
WHERE state NOT IN ('idle')
ORDER BY state_duration DESC
LIMIT 20;"
```

### Step 2 — Check PgBouncer pool state

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
```

### Step 3 — Increase pool size temporarily

In `/etc/pgbouncer/pgbouncer.ini`:
```ini
default_pool_size = 30   # increase from 20
reserve_pool_size = 10   # increase reserve
```

Then reload (no restart needed):
```bash
sudo systemctl reload pgbouncer
```

### Step 4 — Check `query_wait_timeout`

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW CONFIG;" | grep query_wait_timeout
```

If set too low, applications will receive errors before PgBouncer can clear the queue. If set too high (or 0), clients can wait indefinitely.

---

## Firewall Blocking Exporter Port 9127

**Symptom:** Prometheus target DOWN for `pgbouncer_exporter` but exporter is running locally.

**Fix:**
```bash
# On PG17 Host — verify exporter is listening on 0.0.0.0
ss -tlnp | grep 9127

# On PG17 Host — allow Observability Server
sudo ufw allow from <OBSERVABILITY_SERVER_IP> to any port 9127 proto tcp

# From Observability Server — test
nc -zv <PG17_HOST_IP> 9127
```

---

## systemd Service Failures

```bash
# View detailed failure reason
sudo systemctl status pgbouncer-exporter
sudo journalctl -u pgbouncer-exporter --since "5 minutes ago" --no-pager

# Check if service user exists
id pgb_exporter

# Check .env file
sudo ls -la /etc/pgbouncer-exporter/.env
```
