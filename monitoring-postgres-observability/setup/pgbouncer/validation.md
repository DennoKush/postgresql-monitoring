# PgBouncer Validation

Run these checks after completing the installation in `README.md`.

---

## 1. Service Status

```bash
sudo systemctl status pgbouncer
```

Expected: `Active: active (running)`

---

## 2. PgBouncer Listening on Port 6432

```bash
ss -tlnp | grep 6432
```

Expected: a line showing `pgbouncer` listening on `127.0.0.1:6432` (or `0.0.0.0:6432` if exposed).

---

## 3. Admin Database Reachable

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW VERSION;"
```

Expected output example:
```
 version
---------------------------
 PgBouncer 1.22.0
```

---

## 4. SHOW POOLS — Confirm Pools Exist

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW POOLS;"
```

You should see at least one row for the `pgbouncer` admin database and one for each configured application database.

Key columns to check:
- `cl_active` — active client connections
- `cl_waiting` — clients waiting for a server (should be 0 at baseline)
- `sv_active` — active server connections
- `sv_idle` — idle server connections

---

## 5. SHOW CLIENTS

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW CLIENTS;"
```

---

## 6. SHOW SERVERS

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW SERVERS;"
```

---

## 7. SHOW CONFIG

```bash
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW CONFIG;"
```

Verify `max_client_conn`, `default_pool_size`, and `pool_mode` match your intended configuration.

---

## 8. Application Connection Test

Test that an application user can connect through PgBouncer:

```bash
psql -h 127.0.0.1 -p 6432 -U appuser -d mydb -c "SELECT current_user, current_database();"
```

---

## 9. Logs

```bash
sudo journalctl -u pgbouncer -n 30 --no-pager
# Or tail the log file directly
sudo tail -f /var/log/pgbouncer/pgbouncer.log
```

---

## Validation Checklist

- [ ] `systemctl status pgbouncer` shows `active (running)`
- [ ] Port 6432 is listening
- [ ] Admin database reachable with `pgb_exporter` user
- [ ] `SHOW POOLS` returns expected databases
- [ ] `SHOW CONFIG` shows correct `max_client_conn` and `default_pool_size`
- [ ] Application user can connect through PgBouncer
- [ ] No ERROR lines in logs
