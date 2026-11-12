---
name: troubleshoot
description: >
  Diagnose and fix problems on a running Coder deployment as an
  admin. Use when the user says Coder is down, slow, or broken, a
  workspace will not build or start, an agent is stuck connecting,
  SSH or apps do not work, provisioners are offline, logins fail, or
  the health page shows warnings. Do not use for first-time installs
  (use the setup skill), upgrades (use the upgrade skill), or
  template authoring (use the templates skill).
---

# Troubleshoot

Find out why a running Coder deployment misbehaves, fix it with the
user's approval, and confirm the fix. Work from evidence, not from
guesses.

## Source of truth

Read <https://coder.com/docs/llms.txt> for the docs index and pull
the admin troubleshooting pages that match the symptom. Apply the
docs yourself. `coder <subcommand> --help` is authoritative for
flags.

## User interaction

- Ask what they saw and when it started, in one short question.
- Explain findings in plain English: what is wrong, why, and the fix.
- Confirm before any change that restarts Coder, deletes data,
  rebuilds workspaces, or edits deployment config.
- Never print session tokens, database URLs with passwords, OAuth
  secrets, or provisioner keys. Redact them from logs you quote.

## Preconditions

1. `coder whoami`: the user must be signed in as an Owner or
   auditor. If not, ask them to run `coder login <url>`.
2. If `CODER_AGENT_TOKEN` or `CODER_WORKSPACE_NAME` is set, you are
   inside a Coder workspace. Never `pkill coder` or stop the
   workspace you run in.
3. Find the supervisor that runs the server: systemd
   (`systemctl status coder`), Docker compose (`docker compose ps`),
   Helm (`helm list -A | grep coder`), or a PID file from setup under
   `${XDG_STATE_HOME:-$HOME/.local/state}/coder-install/`.

## Triage, in order

Stop at the first layer that fails.

1. Server up: `curl -fsS "$CODER_URL/healthz"`.
2. Deployment health: `GET /api/v2/debug/health` with the session
   token (`coder` CLI config holds it; do not print it). Read each
   section: database, access URL, websocket, DERP, workspace
   proxies, provisioner daemons.
3. Server logs from the supervisor. Look for the first error, not the
   last.
4. Provisioners: `coder provisioner list -o json`. Builds hang in
   `pending` when none are online or tags do not match.
5. The failing workspace:
   - `coder list -o json` for status.
   - `coder show <workspace>` for agent state.
   - Build logs: `coder logs <workspace>` or the build in the UI.
6. Networking from the client: `coder netcheck`, then
   `coder ping <workspace>`. Direct versus DERP tells you if
   peer-to-peer is blocked.
7. Support bundle when the cause is still unclear:
   `coder support bundle <workspace> --yes`. Tell the user where it
   was written; it may contain sensitive data.

## Common causes

| Symptom | Usual cause | First check |
| --- | --- | --- |
| Agent stuck connecting | Workspace cannot reach the access URL | `curl` the access URL from inside the workspace |
| Build pending forever | No matching provisioner | `coder provisioner list` |
| Apps 404 or redirect loop | Wildcard URL or proxy misconfig | Wildcard DNS and the proxy's Host header |
| SSH slow or drops | DERP only, UDP blocked | `coder ping` shows relay |
| Logins fail after OIDC change | Wrong redirect URL or claims | Server logs at login time |
| Server will not start | Database URL or migration error | First error in server logs |

## Fix and verify

1. State the fix in one sentence and get approval.
2. Apply it through the same supervisor that runs Coder.
3. Re-run the triage step that failed, then the steps after it.
4. Hand off: what was wrong, what changed, and how you verified it.
   No CLI walkthrough for the user.

## Safeguards

- Do not restart Coder during business hours without asking.
- Do not delete workspaces, volumes, or database rows to "reset".
- Do not disable TLS, auth, or telemetry to make an error go away.
