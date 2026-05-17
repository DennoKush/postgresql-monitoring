# Log Signals — Failed Database Connections

This document describes the PostgreSQL log patterns that serve as the signal source for the Failed Database Connections monitoring goal.

All signals originate from the PostgreSQL log file. The `log_line_prefix` format configured in `setup/postgres-logging/postgresql.conf.logging` produces lines with this structure:

```
<timestamp> [<pid>]: [<line>] user=<user>,db=<db>,app=<app>,client=<ip> FATAL:  <message>
```

---

## Signal Patterns

### 1. Authentication Failure

**Pattern:** `FATAL: password authentication failed for user`

**Example log line:**
```
2025-05-17 14:23:01 UTC [9102]: [1-1] user=appuser,db=myapp,app=psql,client=10.0.1.5 FATAL:  password authentication failed for user "appuser"
```

**Meaning:** The client supplied an incorrect password (or the SCRAM negotiation failed). The role exists in PostgreSQL, but the credential is wrong.

**Common causes:**
- A new deployment with outdated credentials
- A secret rotation that was not fully propagated
- A credential-stuffing or brute-force attempt
- PgBouncer passing the wrong password to PostgreSQL

**LogQL filter:** `|= "password authentication failed"`

---

### 2. Unknown Role

**Pattern:** `FATAL: role "<user>" does not exist`

**Example log line:**
```
2025-05-17 14:25:00 UTC [9210]: [1-1] user=olduser,db=myapp,app=myapp,client=10.0.1.5 FATAL:  role "olduser" does not exist
```

**Meaning:** The PostgreSQL role (user) named in the connection string does not exist in the database cluster.

**Common causes:**
- Application connecting with a username from a previous environment (e.g., dev → prod misconfiguration)
- A role was dropped but the application was not updated
- Typo in the connection string username
- A migration that created a role under a different name than expected

**LogQL filter:** `|~ "role \".+\" does not exist"`

---

### 3. Unknown Database

**Pattern:** `FATAL: database "<database>" does not exist`

**Example log line:**
```
2025-05-17 14:26:30 UTC [9318]: [1-1] user=appuser,db=wrongdb,app=myapp,client=10.0.1.5 FATAL:  database "wrongdb" does not exist
```

**Meaning:** The database named in the connection string does not exist on this PostgreSQL instance.

**Common causes:**
- Wrong database name in the connection string
- Database was dropped or renamed without updating the application
- Application connecting to the wrong PostgreSQL host (e.g., production app pointing at staging)

**LogQL filter:** `|~ "database \".+\" does not exist"`

---

### 4. No pg_hba.conf Entry

**Pattern:** `FATAL: no pg_hba.conf entry for host "<ip>", user "<user>", database "<db>", SSL <on|off>`

**Example log line:**
```
2025-05-17 14:27:45 UTC [9401]: [1-1] user=appuser,db=myapp,app=myapp,client=192.168.5.10 FATAL:  no pg_hba.conf entry for host "192.168.5.10", user "appuser", database "myapp", SSL off
```

**Meaning:** PostgreSQL evaluated its `pg_hba.conf` rules and found no rule that allows this combination of host IP, username, database, and SSL mode.

**Common causes:**
- A new application host or IP range that has not been added to `pg_hba.conf`
- An existing host's IP address changed (DHCP renewal, server migration)
- An application that was recently migrated to a new subnet
- SSL mode mismatch (application connects with `sslmode=disable` but `pg_hba.conf` requires SSL)

**LogQL filter:** `|= "no pg_hba.conf entry"`

---

### 5. pg_hba.conf Explicit Rejection

**Pattern:** `FATAL: connection rejected: pg_hba.conf rejects connection for host`

**Example log line:**
```
2025-05-17 14:28:10 UTC [9450]: [1-1] user=baduser,db=postgres,app=psql,client=10.0.2.99 FATAL:  connection rejected: pg_hba.conf rejects connection for host "10.0.2.99", user "baduser", database "postgres"
```

**Meaning:** A `reject` rule in `pg_hba.conf` explicitly denied the connection. This is different from "no entry" — there is an entry, and it says `reject`.

**Common causes:**
- Intentional deny rules (blocklists) for known-bad IPs or users
- Overly broad `reject` rules catching legitimate traffic
- A deny-all catch-all at the bottom of `pg_hba.conf`

**LogQL filter:** `|= "connection rejected"`

---

### 6. SSL / Certificate Errors

**Patterns:**
- `FATAL: unsupported frontend protocol`
- `FATAL: SSL connection has been closed unexpectedly`
- `FATAL: could not accept SSL connection`
- Log lines containing `SSL` or `certificate` with severity FATAL or ERROR

**Example log line:**
```
2025-05-17 14:29:00 UTC [9511]: [1-1] user=[unknown],db=[unknown],app=[unknown],client=10.0.1.8 FATAL:  unsupported frontend protocol 1234.5679: server supports 2.0 to 3.0
```

**Meaning:** A TLS negotiation failure between client and server. The client may not support the server's TLS version, or certificates are expired or mismatched.

**Common causes:**
- Client library does not support TLS 1.3 (or whatever version PostgreSQL requires)
- Expired server or client certificate
- `sslmode=require` on the client but PostgreSQL has SSL disabled
- Self-signed certificate not trusted by the client

**LogQL filter:** `|~ "(?i)SSL|certificate"` combined with `|= "FATAL"`

---

## Severity Distribution

| PostgreSQL Severity | Meaning | Included? |
|---|---|---|
| `PANIC` | Server is shutting down | Yes |
| `FATAL` | Connection or session terminated | Yes — primary signal |
| `ERROR` | Error recoverable within the session | Yes |
| `WARNING` | Non-fatal advisory | Yes |
| `LOG` | Informational (e.g., connection established) | No — dropped by Alloy pipeline |
| `INFO` | Informational | No — dropped |
| `DEBUG` | Debug output | No — dropped |

The Alloy pipeline in `config.alloy` drops `LOG`, `INFO`, and `DEBUG` lines before shipping to Loki to reduce ingest volume.

---

## Log Line Anatomy

Given the configured `log_line_prefix`:

```
%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h
```

A full failed connection line looks like:

```
2025-05-17 14:23:01 UTC [9102]: [1-1] user=appuser,db=myapp,app=psql,client=10.0.1.5 FATAL:  password authentication failed for user "appuser"
```

| Field | Value | Label in Loki |
|---|---|---|
| `%t` | `2025-05-17 14:23:01 UTC` | Raw timestamp |
| `%p` | `9102` | Not labeled (high cardinality) |
| `%l` | `1` | Not labeled |
| `user=` | `appuser` | Not labeled (high cardinality) — extracted for log content search |
| `db=` | `myapp` | Not labeled — use LogQL line filter to search |
| `app=` | `psql` | Not labeled |
| `client=` | `10.0.1.5` | Not labeled (high cardinality) |
| `FATAL` | severity | Labeled as `severity=FATAL` |
| Error message | `password authentication failed` | Normalized as `error_type=auth_failed` |

The deliberate choice not to label `user`, `db`, or `client` avoids Loki index cardinality explosions. Use line filters (`|=`, `|~`) to search within log content for specific users or IPs.
