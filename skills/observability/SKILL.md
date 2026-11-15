---
name: observability
description: >
  Set up and use monitoring for a Coder deployment: Prometheus
  metrics, server, provisioner, and agent logs, the deployment health
  check, and alerts. Use when the user asks to monitor Coder, expose
  or scrape metrics, build Grafana dashboards, pick alert rules, read
  the health page or a health error code (EACS, EDB, EDERP, ESTUN,
  EWS, EWP, EPD), find where logs go, or set up admin notifications.
  Do not use for fixing a specific outage (use the troubleshoot
  skill) or load testing (use the scale-testing skill).
---

# Observability

Make a Coder deployment observable: metrics scraped, logs kept, the
health check read, and a few alerts that catch real problems early.

## Source of truth

Read <https://coder.com/docs/llms.txt>, then as Markdown under
`https://coder.com/docs/`: `admin/integrations/prometheus.md` (full
metric list), `admin/monitoring/logs.md`,
`admin/monitoring/health-check.md`,
`admin/monitoring/notifications.md`, and the observability section of
`tutorials/best-practices/scale-coder.md`. `coder server --help` is
authoritative for flags.

## User interaction

- Ask what they already run: Prometheus, Grafana, Loki, Datadog, or
  nothing.
- If they run nothing on Kubernetes, suggest the Coder observability
  Helm chart at <https://github.com/coder/observability>. It bundles
  Prometheus, Grafana, Alertmanager, dashboards, and alerts.
- Confirm before you change server config; it needs a restart.

## Metrics

1. Turn on the endpoint: `CODER_PROMETHEUS_ENABLE=true` (or
   `--prometheus-enable`). It listens on `127.0.0.1:2112` by
   default; set `CODER_PROMETHEUS_ADDRESS` to change it. The Helm
   chart sets `0.0.0.0:2112` once enabled.
2. Optional collectors:
   - `CODER_PROMETHEUS_COLLECT_DB_METRICS=true` for database metrics.
   - `CODER_PROMETHEUS_COLLECT_AGENT_STATS=true` for
     `coderd_agentstats_*`. At large scale, limit labels with
     `CODER_PROMETHEUS_AGGREGATE_AGENT_STATS_BY=agent_name`.
3. Check it from the server host: `curl -fsS http://localhost:2112/`.
4. Scrape it. On Kubernetes the chart exposes no Service for metrics;
   create a headless Service on port 2112 and a `ServiceMonitor`, as
   in the Prometheus page. Elsewhere add a `scrape_config` target.
5. External provisioners serve their own metrics with the same
   `--prometheus-enable` and `--prometheus-address` flags on
   `coder provisioner start`. Scrape them separately.

## Logs

- `coderd` logs to standard output in human format. Read them with
  `kubectl logs deployment/coder -n <namespace>` or
  `journalctl -u coder`.
- For a log store, write JSON logs to a file with
  `CODER_LOGGING_JSON=<path>`. Set `CODER_LOGGING_HUMAN=/dev/null` to
  turn off the human format; an empty string does not.
- `CODER_LOG_FILTER=.*` turns on debug logs. Use it briefly.
- Audit log entries, user activity, SSO and OIDC logs, and server
  errors are in the `coderd` logs.
- Agent logs are in `/tmp/coder-agent.log` in Linux and macOS
  workspaces, truncated at 5 MB. Startup script logs are in the same
  temp directory.
- On Kubernetes, `coder-logstream-kube` streams pod events into the
  workspace build log.
- Keep logs at least 30 days, ideally 90.

## Health check

- Dashboard: `<access-url>/health`. JSON:
  `GET /api/v2/debug/health` with an Owner session token.
- Each section has error codes, for example `EACS03` (cannot fetch
  `/healthz`), `EDB02` (database latency high), `EDERP02` (unhealthy
  DERP nodes), `ESTUN02` (hard NAT), `EWS01` (WebSocket failed), and
  `EPD01` (no provisioner daemons). Look up the code on the health
  check page for the fix.

## Alerts

Start with a few alerts on metrics from the Prometheus page:

| Signal | Metric |
| --- | --- |
| API tail latency | `coderd_api_request_latencies_seconds` |
| Database tail latency | `coderd_db_query_latencies_seconds` |
| Build queue wait | `coderd_provisioner_job_queue_wait_seconds` |
| Provisioners online | `coderd_provisionerd_num_daemons` |
| Failed builds | `coderd_workspace_builds_total` by `status` |
| License problems | `coderd_license_errors`, `coderd_license_warnings` |

Also alert on CPU and memory of `coderd`, provisioners, and
PostgreSQL from your infrastructure metrics. Watch daily and weekly
peaks. Keep metrics at least six months to see trends.

## Notifications

Coder delivers notifications to the dashboard Inbox and by SMTP or
webhook (`CODER_NOTIFICATIONS_METHOD`). Template admins get a weekly
report of failed builds. Workspace owners get build failures and,
when the template sets up resource monitoring, out-of-memory and
out-of-disk alerts. Send a test with `coder notifications test`.

## Safeguards

- Do not expose port 2112 to the internet.
- Turn debug log filters off again after you finish debugging.
- Do not print session tokens when you call the health API.
