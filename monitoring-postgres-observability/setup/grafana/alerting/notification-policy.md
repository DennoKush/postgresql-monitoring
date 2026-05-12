# Grafana Notification Policy

Notification policies in Grafana control which contact point receives which alerts. This document configures routing so that PostgreSQL and PgBouncer alerts go to the Microsoft Teams channel.

---

## Default Policy vs Custom Routes

Grafana has a **default policy** that sends all alerts to the default contact point. You can add **routes** to override routing for specific label matchers.

---

## Step 1 — Set Default Contact Point

1. Go to **Alerting → Notification policies**
2. Click **Edit** on the **Default policy**
3. Set **Default contact point** to `Microsoft Teams — PostgreSQL Alerts`
4. Click **Update default policy**

This sends all alerts to Teams unless a more specific route matches.

---

## Step 2 — Add a Specific Route for PostgreSQL Alerts (optional)

If you want to route only PostgreSQL/PgBouncer alerts to Teams and leave other alerts to a different contact point:

1. Under the default policy, click **Add child policy**
2. Set **Matching labels:**
   - `component = postgres` → for connection count alerts
   - Or use `team = dba` to match all DBA-owned alerts
3. Set **Contact point:** `Microsoft Teams — PostgreSQL Alerts`
4. **Continue matching:** off (stops processing after this route matches)
5. Click **Save policy**

Repeat for PgBouncer:
- Matching label: `component = pgbouncer`
- Contact point: `Microsoft Teams — PostgreSQL Alerts`

---

## Step 3 — Configure Grouping and Timing

Under the contact point route, configure:

| Setting | Recommended Value | Notes |
|---|---|---|
| Group by | `alertname`, `instance` | Prevents alert flood; one message per alert per instance |
| Group wait | `30s` | Wait this long before sending the first notification |
| Group interval | `5m` | How often to re-notify for the same group |
| Repeat interval | `1h` | How long between reminders for a sustained alert |

---

## Step 4 — Silence Policies (optional)

To suppress alerts during maintenance windows:

1. Go to **Alerting → Silences**
2. Click **Add Silence**
3. Set matching labels: `instance = pg18-primary`
4. Set a time range for the maintenance window
5. Click **Submit**

All alerts matching the silence labels will be suppressed for the duration.

---

## Verification

1. Trigger a test alert from a contact point (see [`contact-point-teams.md`](contact-point-teams.md)).
2. Verify the message arrives in Teams.
3. Check **Alerting → Alert history** for delivery records.
4. Check **Alerting → Notification policies** to confirm the route is active and linked correctly.
