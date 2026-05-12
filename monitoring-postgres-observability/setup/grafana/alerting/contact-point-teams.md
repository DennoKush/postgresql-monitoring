# Grafana Contact Point — Microsoft Teams

This document explains how to configure Grafana to send alert notifications to a Microsoft Teams channel.

---

## Step 1 — Obtain a Teams Webhook URL

Microsoft Teams now uses **Power Automate Workflows** for webhooks (the legacy "Incoming Webhook" connector is being retired for new deployments, but still works on existing tenants).

### Option A — Power Automate Workflow (Recommended for new setups)

1. In Microsoft Teams, navigate to the channel where you want to receive alerts.
2. Click the **three dots (...)** next to the channel name → **Workflows**.
3. Search for **"Post to a channel when a webhook request is received"**.
4. Click **Add** and follow the setup wizard.
5. Copy the webhook URL provided at the end of the wizard.

### Option B — Legacy Incoming Webhook Connector

1. In Teams, click **Apps** → search for **Incoming Webhook**.
2. Select the channel and click **Add**.
3. Give it a name (e.g., `PostgreSQL Alerts`), optionally upload an icon.
4. Click **Create** and copy the webhook URL.

Keep this URL secret. It grants posting access to the channel.

---

## Step 2 — Add Contact Point in Grafana

1. Log in to Grafana at `http://<OBSERVABILITY_SERVER_IP>:3000`
2. Go to **Alerting → Contact points**
3. Click **Add contact point**
4. Fill in:
   - **Name:** `Microsoft Teams — PostgreSQL Alerts`
   - **Integration:** Select **Microsoft Teams**
   - **URL:** Paste the webhook URL from Step 1

   > The URL is stored encrypted in Grafana's database after saving — it will not appear in plain text again.

5. **Optional — message template:**
   Under **Optional settings**, add a message body:
   ```
   **Alert: {{ template "default.title" . }}**
   
   {{ template "default.message" . }}
   
   Severity: {{ .CommonLabels.severity }}
   Instance: {{ .CommonLabels.instance }}
   ```

6. Click **Test** to send a test message to Teams. Confirm it appears in the channel.
7. Click **Save contact point**.

---

## Step 3 — Test Alert

After saving:

1. Click **Test** on the contact point.
2. Check the Teams channel — a test message should appear within a few seconds.

If no message arrives, see the troubleshooting section below.

---

## Step 4 — Notification Policy

See [`notification-policy.md`](notification-policy.md) for routing PostgreSQL and PgBouncer alerts to this contact point.

---

## Troubleshooting

### Teams channel not receiving alerts

1. **Verify webhook URL is correct:**
   ```bash
   curl -H "Content-Type: application/json" \
     -d '{"text":"Test from curl"}' \
     https://your-teams-webhook-url
   ```
   Expected response: `1` (plain text, not JSON). If you get `400` or `403`, the URL is invalid or expired.

2. **Check Grafana alert state:**
   - Go to **Alerting → Alert Rules** — verify the rule is in Firing state.
   - Go to **Alerting → Contact points** — check the "Last notification" status.

3. **Check Grafana logs:**
   ```bash
   sudo journalctl -u grafana-server -n 50 --no-pager | grep -i "teams\|webhook\|notification"
   ```

4. **Power Automate workflow not triggering:**
   - Check the workflow run history in Power Automate.
   - Ensure the flow is still active (flows can be paused automatically after errors).

### Message formatting issues

Teams uses a subset of Markdown. Avoid complex nested formatting in alert messages. Use simple bold (`**text**`) and line breaks.

If using Adaptive Cards format, ensure Grafana version supports it (Grafana 10.x introduced richer Teams message formatting).
