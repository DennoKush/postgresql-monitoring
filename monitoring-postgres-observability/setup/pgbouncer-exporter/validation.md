# pgbouncer_exporter Validation

Run these checks after completing the installation in `README.md`.

---

## 1. Service Status

```bash
sudo systemctl status pgbouncer-exporter
```

Expected: `Active: active (running)`

---

## 2. Metrics Endpoint — Up Status

```bash
curl -s http://localhost:9127/metrics | grep pgbouncer_up
```

Expected:
```
pgbouncer_up 1
```

`pgbouncer_up 0` means the exporter is running but cannot connect to PgBouncer. Check credentials and PgBouncer status.

---

## 3. Pool Metrics Present

```bash
curl -s http://localhost:9127/metrics | grep pgbouncer_pools_ | head -20
```

Expected (example):
```
pgbouncer_pools_cl_active{database="mydb",user="appuser"} 0
pgbouncer_pools_cl_waiting{database="mydb",user="appuser"} 0
pgbouncer_pools_sv_active{database="mydb",user="appuser"} 0
pgbouncer_pools_sv_idle{database="mydb",user="appuser"} 2
```

If you only see `pgbouncer` admin pool rows and no application pools, that means no application has connected through PgBouncer yet. Make a test connection first.

---

## 4. Config Metrics Present

```bash
curl -s http://localhost:9127/metrics | grep pgbouncer_config_
```

Expected:
```
pgbouncer_config_max_client_conn 200
pgbouncer_config_default_pool_size 20
```

---

## 5. Remote Reachability

Run from the **Observability Server**:

```bash
curl -s http://<PG18_HOST_IP>:9127/metrics | grep pgbouncer_up
```

Expected: `pgbouncer_up 1`

---

## 6. Logs

```bash
sudo journalctl -u pgbouncer-exporter -n 30 --no-pager
```

---

## Validation Checklist

- [ ] `systemctl status pgbouncer-exporter` shows `active (running)`
- [ ] `pgbouncer_up 1` returned from `/metrics`
- [ ] `pgbouncer_pools_cl_waiting` metric present
- [ ] `pgbouncer_pools_sv_active` and `pgbouncer_pools_sv_idle` metrics present
- [ ] `pgbouncer_config_max_client_conn` metric present
- [ ] Metrics endpoint reachable from Observability Server
