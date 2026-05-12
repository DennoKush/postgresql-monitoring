# Teams Webhook Setup

This document covers two methods for creating a webhook URL to receive Grafana alerts in a Microsoft Teams channel.

---

## Method A — Power Automate Workflow (Current/Recommended)

Microsoft is migrating away from the legacy Incoming Webhook connector. Use this method for new deployments.

### Steps

1. Open Microsoft Teams and navigate to the **target channel** (not the team — the specific channel).

2. Click the **three dots (…)** next to the channel name → **Workflows**.

3. In the search box, type:
   ```
   Post to a channel when a webhook request is received
   ```

4. Select the template and click **Add**.

5. Follow the wizard:
   - Name the workflow (e.g., `PostgreSQL Alert Notifications`)
   - Confirm the Teams channel shown is correct
   - Click **Next** then **Add workflow**

6. After the workflow is created, you will see a URL like:
   ```
   https://prod-XX.REGION.logic.azure.com:443/workflows/...
   ```
   Copy this URL. This is your webhook URL.

7. Store the URL securely — treat it as a secret.

### Test the Webhook

```bash
curl -H "Content-Type: application/json" \
     -d '{"text": "Test message from ProjectClearSight setup"}' \
     "https://prod-XX.REGION.logic.azure.com:443/workflows/..."
```

Expected response: `{"$content-type":"application/json","$content":"..."}` or `Accepted`

Check the Teams channel — the test message should appear within a few seconds.

---

## Method B — Legacy Incoming Webhook Connector

Use only if your Teams tenant still supports this method (older tenants or specific admin settings).

### Steps

1. Go to the **target channel** in Teams.

2. Click the **three dots (…)** next to the channel name → **Connectors**.

3. Search for **"Incoming Webhook"** and click **Configure**.

4. Enter a name (e.g., `PostgreSQL Monitoring`) and optionally upload a custom image.

5. Click **Create**.

6. Copy the webhook URL shown (format: `https://TENANTNAME.webhook.office.com/...`).

7. Click **Done**.

### Test the Webhook

```bash
curl -H "Content-Type: application/json" \
     -d '{"text": "Test message from ProjectClearSight setup"}' \
     "https://TENANTNAME.webhook.office.com/..."
```

Expected response: `1`

---

## Configuring the URL in Grafana

Once you have the URL, add it as a Grafana contact point:

1. Grafana UI → **Alerting → Contact points → Add contact point**
2. Integration: **Microsoft Teams**
3. URL: paste the webhook URL
4. Click **Test** to verify delivery
5. Click **Save contact point**

See [`../grafana/alerting/contact-point-teams.md`](../grafana/alerting/contact-point-teams.md) for full details.

---

## Common Delivery Problems

| Problem | Likely Cause | Fix |
|---|---|---|
| No message in Teams | Wrong or expired webhook URL | Re-create the workflow/connector and get a fresh URL |
| `400 Bad Request` from curl test | Malformed JSON payload | Validate JSON with `echo '...' \| python3 -m json.tool` |
| `403 Forbidden` | URL belongs to wrong tenant or revoked | Re-create the webhook in the correct Teams tenant |
| Messages appear but alerts don't | Notification policy misconfigured | Check **Alerting → Notification policies** in Grafana |
| Power Automate flow paused | Azure Logic Apps auto-paused due to errors | Re-enable the flow in Power Automate portal |
| Alert fires but no notification | Alert state is Pending, not Firing | Check the `for` duration in the alert rule — alert must sustain before firing |
