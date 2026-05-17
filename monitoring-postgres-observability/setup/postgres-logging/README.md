# PostgreSQL Logging Configuration

This guide configures PostgreSQL to emit structured log lines that Grafana Alloy can tail and forward to Loki. The focus is failed connection events — authentication failures, missing roles, missing databases, and pg_hba.conf rejections.

This setup is a prerequisite for the **Failed Database Connections** monitoring goal.

---

## 1. Verify the Current Log Destination

```bash
sudo -u postgres psql -c "SHOW log_destination;"
sudo -u postgres psql -c "SHOW logging_collector;"
sudo -u postgres psql -c "SHOW log_directory;"
sudo -u postgres psql -c "SHOW log_filename;"
```

Note the values — you will need the full log path for the Alloy configuration.

---

## 2. Apply the Logging Configuration

Copy the logging snippet into your `postgresql.conf`:

```bash
sudo -u postgres psql -c "SHOW config_file;"
# Typical Ubuntu path: /etc/postgresql/<version>/main/postgresql.conf
```

Append or update the following settings. The full parameter reference is in [`postgresql.conf.logging`](postgresql.conf.logging).

```bash
sudo nano /etc/postgresql/<version>/main/postgresql.conf
```

Settings to add or update:

```ini
# --- Connection logging ---
log_connections    = on
log_disconnections = on
log_hostname       = off

# --- Collector ---
logging_collector  = on
log_directory      = 'log'
log_filename       = 'postgresql-%Y-%m-%d.log'
log_rotation_age   = 1d
log_rotation_size  = 100MB
log_truncate_on_rotation = on

# --- Line format ---
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# --- Minimum severity to log ---
log_min_messages         = warning
log_min_error_statement  = error
```

**Why `log_hostname = off`?** Reverse-DNS lookups add latency to connection setup and produce high-cardinality hostname labels in Loki. Use `client=%h` (IP address) instead.

**Why `log_line_prefix` with this format?** The prefix embeds timestamp, PID, user, database, application name, and client IP in every line — making LogQL label extraction straightforward without parsing free-form text.

---

## 3. Reload PostgreSQL

A reload is sufficient for these settings — no restart required:

```bash
sudo systemctl reload postgresql
```

Verify the reload succeeded:

```bash
sudo systemctl status postgresql
```

---

## 4. Confirm Log Output

```bash
# Find the active log file
sudo -u postgres psql -c "SELECT pg_current_logfile();"

# Tail it and attempt a bad connection in another terminal
sudo tail -f /var/lib/postgresql/<version>/main/log/postgresql-$(date +%Y-%m-%d).log
```

In a second terminal, trigger a failed connection to generate a test log line:

```bash
psql -h 127.0.0.1 -U nonexistent_user -d postgres
```

Expected log output:

```
2025-05-17 12:00:00 UTC [1234]: [1-1] user=nonexistent_user,db=postgres,app=psql,client=127.0.0.1 FATAL:  role "nonexistent_user" does not exist
```

---

## 5. Note the Full Log Path

Record the absolute path to the log directory — you will supply it to Alloy as `<POSTGRES_LOG_PATH>`:

```bash
sudo -u postgres psql -c "SELECT current_setting('data_directory') || '/' || current_setting('log_directory') AS log_path;"
```

Common Ubuntu path:

```
/var/lib/postgresql/<version>/main/log
```

---

## Validation Checklist

- [ ] `log_connections = on` confirmed in `postgresql.conf`
- [ ] `logging_collector = on` confirmed
- [ ] `log_line_prefix` includes `user=`, `db=`, `client=` fields
- [ ] Failed connection attempt produces a `FATAL` log line in the log file
- [ ] Full log path recorded for Alloy configuration

---

## Troubleshooting

### No log file appears after reload

```bash
sudo -u postgres psql -c "SHOW logging_collector;"
```

If it returns `off`, the setting requires a **PostgreSQL restart** (not just a reload) to take effect:

```bash
sudo systemctl restart postgresql
```

### Logs are written but missing the expected fields

Verify `log_line_prefix` was saved correctly:

```bash
sudo -u postgres psql -c "SHOW log_line_prefix;"
```

If it does not match, re-edit `postgresql.conf`, confirm there are no syntax errors, and reload.

### Log directory is not writable

```bash
ls -ld /var/lib/postgresql/<version>/main/log
```

The directory must be owned by the `postgres` user. If it is missing:

```bash
sudo mkdir -p /var/lib/postgresql/<version>/main/log
sudo chown postgres:postgres /var/lib/postgresql/<version>/main/log
sudo chmod 700 /var/lib/postgresql/<version>/main/log
sudo systemctl restart postgresql
```
