# Microsoft Teams Integration

This directory contains documentation for integrating Grafana alerts with Microsoft Teams.

## Files

- [`webhook-setup.md`](webhook-setup.md) — Step-by-step instructions for creating a Teams webhook URL (both legacy Incoming Webhook and Power Automate Workflow methods).

## Where This Fits in the Stack

```
Grafana Alert (firing)
        │
        │  HTTP POST — JSON payload
        ▼
Microsoft Teams Webhook URL
        │
        │
        ▼
Teams Channel Message
```

Grafana sends a POST request with alert details to the webhook URL. Teams renders this as a channel message. No polling or agent is required on the Teams side.

## Prerequisites

- A Microsoft Teams channel where alerts should appear
- Owner or member permissions in the Teams team (to add connectors or workflows)
- The webhook URL configured in Grafana as a Contact Point

See [`../grafana/alerting/contact-point-teams.md`](../grafana/alerting/contact-point-teams.md) for how to configure the Teams contact point in Grafana.
