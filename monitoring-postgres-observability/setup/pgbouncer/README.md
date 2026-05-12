# PgBouncer Setup

PgBouncer is a lightweight connection pooler for PostgreSQL. Applications connect to PgBouncer on port **6432**; PgBouncer maintains a smaller pool of actual PostgreSQL connections and multiplexes client requests across them.

---

## 1. Install PgBouncer

```bash
sudo apt update
sudo apt install -y pgbouncer

# Verify installation
pgbouncer --version
```

---

## 2. Create a PgBouncer PostgreSQL User

PgBouncer needs to authenticate application users against PostgreSQL. Create a user that PgBouncer will use to proxy connections (or use the application user directly depending on pool mode).

```sql
-- Connect as postgres superuser
sudo -u postgres psql

-- Create application user (if not already existing)
CREATE USER appuser WITH PASSWORD '<app_user_password>';
GRANT CONNECT ON DATABASE mydb TO appuser;

-- Create pgbouncer_exporter admin user (for monitoring)
CREATE USER pgb_exporter WITH PASSWORD '<pgb_exporter_password>';
```

---

## 3. Configure pgbouncer.ini

```bash
sudo cp /path/to/this/repo/setup/pgbouncer/pgbouncer.ini.example /etc/pgbouncer/pgbouncer.ini
sudo nano /etc/pgbouncer/pgbouncer.ini
```

Key parameters to configure:
- `[databases]` section: map each application database to the PostgreSQL host/port.
- `listen_addr` and `listen_port`: where PgBouncer accepts client connections.
- `auth_file`: path to `userlist.txt`.
- `pool_mode`: typically `transaction` for most web applications.
- `max_client_conn`: upper limit on total client connections.
- `default_pool_size`: PostgreSQL server connections per pool.

---

## 4. Configure userlist.txt

```bash
sudo cp /path/to/this/repo/setup/pgbouncer/userlist.txt.example /etc/pgbouncer/userlist.txt
sudo nano /etc/pgbouncer/userlist.txt
```

Add the application user and the pgbouncer_exporter admin user. Passwords can be md5-hashed or plain (plain is acceptable on secured servers; md5 is safer):

```bash
# Generate md5 password hash: "md5" + md5(password + username)
echo -n "md5$(echo -n '<password><username>' | md5sum | awk '{print $1}')"
```

---

## 5. Set Permissions and Start PgBouncer

```bash
sudo chown postgres:postgres /etc/pgbouncer/pgbouncer.ini
sudo chown postgres:postgres /etc/pgbouncer/userlist.txt
sudo chmod 640 /etc/pgbouncer/pgbouncer.ini
sudo chmod 640 /etc/pgbouncer/userlist.txt

sudo systemctl enable pgbouncer
sudo systemctl start pgbouncer
```

---

## 6. PostgreSQL pg_hba.conf

PgBouncer connects to PostgreSQL from `127.0.0.1`. Ensure PostgreSQL accepts connections from PgBouncer users:

```
# /etc/postgresql/17/main/pg_hba.conf
host    mydb        appuser         127.0.0.1/32    scram-sha-256
host    pgbouncer   pgb_exporter    127.0.0.1/32    scram-sha-256
```

Reload PostgreSQL after changes:

```bash
sudo systemctl reload postgresql
```

---

## 7. Validate

See [`validation.md`](validation.md).

Quick check:

```bash
sudo systemctl status pgbouncer

# Connect to PgBouncer admin database
psql -h 127.0.0.1 -p 6432 -U pgb_exporter pgbouncer -c "SHOW VERSION;"
```

---

## Troubleshooting

### PgBouncer fails to start

```bash
sudo journalctl -u pgbouncer -n 50 --no-pager
```

Common causes:
- Syntax error in `pgbouncer.ini` — verify with `pgbouncer -d -v /etc/pgbouncer/pgbouncer.ini`
- Port 6432 already in use — check with `ss -tlnp | grep 6432`
- Wrong ownership on config files

### PgBouncer admin database not reachable

**Symptom:** `psql: error: connection to server at "127.0.0.1", port 6432 failed: FATAL: no such user: pgb_exporter`

**Fix:**
1. Confirm `admin_users = pgb_exporter` and `stats_users = pgb_exporter` are set in `pgbouncer.ini`.
2. Confirm `pgb_exporter` entry exists in `userlist.txt`.
3. Reload PgBouncer: `sudo systemctl reload pgbouncer`

### PgBouncer SHOW commands failing

**Symptom:** `ERROR: permission denied`

**Fix:** The connecting user must be listed in both `admin_users` (full admin) or `stats_users` (SHOW commands only) in `pgbouncer.ini`.

### Pool exhaustion — clients stuck waiting

This is a monitoring signal, not a bug. See [`monitoring-goals/connection-pool-exhaustion/README.md`](../../monitoring-goals/connection-pool-exhaustion/README.md) for how to detect and respond.
