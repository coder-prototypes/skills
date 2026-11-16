---
name: analytics
description: >
  Report how a Coder deployment is used: active users, template and app
  usage, connection latency, user growth, and AI Gateway sessions. Use
  when an admin asks who is using Coder, which templates or IDEs are
  popular, how adoption is trending, or how much AI usage there is. Reads
  the insights API from an admin toolbox workspace; it changes nothing.
---

# Deployment analytics

The insights API needs an admin (Owner or Auditor) session. Run the
requests from the `admin-toolbox` workspace described in the
support-bundle skill: its `CODER_URL` and `CODER_SESSION_TOKEN` already
point at this deployment as the workspace owner. Set it up first if the
chat has no toolbox workspace.

## 1. Pick the window

Ask for the period if the user did not say one; default to the last 7
days. The API is strict about times:

- `start_time` must be midnight (`T00:00:00`).
- `end_time` must be midnight, or the start of the current hour when it
  is today. It cannot be in the future.

```sh
start=$(date -u -d '7 days ago' +%Y-%m-%dT00:00:00Z)
end=$(date -u +%Y-%m-%dT%H:00:00Z)
api() { curl -fsS -H "Coder-Session-Token: $CODER_SESSION_TOKEN" "$CODER_URL/api/v2/$1"; }
```

Add `&template_ids=<id>,<id>` to scope a report to templates; get IDs
with `coder_list_templates`.

## 2. Collect

| Question | Request |
|---|---|
| Active users and app usage per template | `api "insights/templates?start_time=$start&end_time=$end&interval=day"` |
| Who used Coder, and for how long | `api "insights/user-activity?start_time=$start&end_time=$end"` |
| Connection latency per user | `api "insights/user-latency?start_time=$start&end_time=$end"` |
| Daily active users | `api "insights/daus?tz_offset=0"` |
| Active, dormant, and suspended users over time | `api "insights/user-status-counts?tz_offset=0"` |
| AI Gateway sessions (Premium) | `api "ai-gateway/sessions?limit=50"` |

Pipe through `jq` to keep only what you need. In the templates report,
`report.apps_usage` lists IDEs and apps by `seconds` and
`interval_reports` gives the per-day series. Use the user's time zone
offset in hours for `tz_offset` when they give one.

## 3. Report

Lead with three to five numbers that answer the question, for example
weekly active users, the top templates by active users, and the most
used IDE. Then give a short table. Compare with the previous period of
the same length when the user asks about a trend, by running the same
request with the earlier window.

Say plainly when data is missing:

- Template and app usage only counts workspaces whose agents report
  stats, so a new deployment shows zeros.
- AI Gateway sessions need AI Gateway and a Premium license; a 404 or
  403 means it is not enabled.

## Rules

- Read only. Never change users, templates, or settings from this skill.
- Do not print `CODER_SESSION_TOKEN`.
- Report usernames only when the user asks for per-user detail.
