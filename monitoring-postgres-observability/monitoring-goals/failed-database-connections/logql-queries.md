# LogQL Queries — Failed Database Connections

Use these queries in the Grafana Explore view (select the Loki datasource), dashboard panels, or alert rule conditions.

The `job="postgres"` label selector assumes Alloy was configured with `job = "postgres"` in the `local.file_match` targets.

---

## Total Failed Database Connections

```logql
# Count of all FATAL lines in the selected time range
sum(count_over_time({job="postgres"} |= "FATAL" [$__range]))

# Rate of FATAL lines per second (for time-series panels)
sum(rate({job="postgres"} |= "FATAL" [$__interval]))

# Count in the last 5 minutes (for alert conditions)
sum(count_over_time({job="postgres"} |= "FATAL" [5m]))
```

---

## Authentication Failures

```logql
# Count of password authentication failures
sum(count_over_time({job="postgres"} |= "password authentication failed" [$__range]))

# Rate over time
sum(rate({job="postgres"} |= "password authentication failed" [$__interval]))

# Count in the last 2 minutes (alert threshold query)
sum(count_over_time({job="postgres"} |= "password authentication failed" [2m]))
```

---

## Unknown Role Errors

```logql
# Count of 'role does not exist' errors
sum(count_over_time({job="postgres"} |~ `role ".+" does not exist` [$__range]))

# Rate over time
sum(rate({job="postgres"} |~ `role ".+" does not exist` [$__interval]))

# Count in the last 5 minutes
sum(count_over_time({job="postgres"} |~ `role ".+" does not exist` [5m]))
```

---

## Unknown Database Errors

```logql
# Count of 'database does not exist' errors
sum(count_over_time({job="postgres"} |~ `database ".+" does not exist` [$__range]))

# Rate over time
sum(rate({job="postgres"} |~ `database ".+" does not exist` [$__interval]))

# Count in the last 5 minutes
sum(count_over_time({job="postgres"} |~ `database ".+" does not exist` [5m]))
```

---

## No pg_hba.conf Entry Errors

```logql
# Count of pg_hba.conf rejection events
sum(count_over_time({job="postgres"} |= "no pg_hba.conf entry" [$__range]))

# Rate over time
sum(rate({job="postgres"} |= "no pg_hba.conf entry" [$__interval]))

# Count in the last 5 minutes
sum(count_over_time({job="postgres"} |= "no pg_hba.conf entry" [5m]))
```

---

## SSL / Certificate Errors

```logql
# Count of SSL and certificate-related FATAL lines
sum(count_over_time({job="postgres"} |= "FATAL" |~ "(?i)SSL|certificate" [$__range]))

# Rate over time
sum(rate({job="postgres"} |= "FATAL" |~ "(?i)SSL|certificate" [$__interval]))
```

---

## Failed Connection Rate Over Time

```logql
# Total FATAL events per 5-minute bucket (for time-series panel)
sum(count_over_time({job="postgres"} |= "FATAL" [5m]))

# Broken out by error_type label (requires Alloy pipeline label extraction)
sum by (error_type) (count_over_time({job="postgres", error_type!=""} [5m]))

# Rate of all FATAL events per second
sum(rate({job="postgres"} |= "FATAL" [$__interval]))
```

---

## Top Failed Connection Messages

```logql
# Count by normalized error_type label (low-cardinality — safe for tables)
sum by (error_type) (count_over_time({job="postgres", error_type!=""} [$__range]))

# Raw log lines of the most recent FATAL events (for the logs panel)
{job="postgres"} |= "FATAL"

# Filter raw log lines to a specific error type without high-cardinality labels
{job="postgres"} |= "FATAL" |= "password authentication failed"
{job="postgres"} |= "FATAL" |~ `role ".+" does not exist`
{job="postgres"} |= "FATAL" |= "no pg_hba.conf entry"
```

---

## Ingestion Health Check

```logql
# Total log lines received in the last 10 minutes — use in LokiPostgresLogIngestionStopped alert
sum(count_over_time({job="postgres"} [10m]))

# Line count per 1-minute bucket — should be non-zero if PostgreSQL is active
sum(count_over_time({job="postgres"} [1m]))
```

---

## Searching by Specific User or IP (Line Filter — No Label Required)

Because usernames and IPs are not promoted as labels, use line filters to search within log content:

```logql
# Find all FATAL lines mentioning a specific username
{job="postgres"} |= "FATAL" |= `user=appuser`

# Find all FATAL lines from a specific client IP
{job="postgres"} |= "FATAL" |= `client=10.0.1.5`

# Authentication failures from a specific IP
{job="postgres"} |= "password authentication failed" |= `client=10.0.2.99`

# All errors involving a specific database name
{job="postgres"} |= "FATAL" |= `db=myapp`
```

Use these in the Grafana Explore view when investigating a specific incident. Do not use them as alert rule queries — they rely on content that may change.

---

## Query Tips

- **`[$__range]`** — uses the Grafana dashboard time range. Best for stat panels showing a count over the selected window.
- **`[$__interval]`** — uses Grafana's calculated step interval. Best for time-series panels showing rate-over-time.
- **`[5m]`** or `[2m]`** — fixed windows. Use these in alert rule conditions where `$__range` is not available.
- **Line filter order matters:** put the most selective filter first (e.g., `|= "FATAL"` before `|= "password authentication failed"`) to minimize lines evaluated by subsequent filters.
- **`|~` uses RE2 regex.** Backtick strings avoid double-escaping: `` |~ `role ".+" does not exist` `` is cleaner than `|~ "role \".+\" does not exist"`.
