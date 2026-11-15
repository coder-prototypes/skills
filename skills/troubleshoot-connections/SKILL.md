---
name: troubleshoot-connections
description: >
  Diagnose and fix connection problems between users, workspaces, and
  a Coder deployment. Use when a workspace agent is stuck
  "connecting", `coder ssh` hangs or drops, port forwarding fails,
  connections are slow or relayed through DERP instead of direct,
  `coder ping` or `coder netcheck` output needs explaining, or
  workspace apps and the dashboard port forwarder do not load. Do not
  use for builds that fail, servers that will not start, or logins
  that fail (use the troubleshoot skill), or for first-time installs
  (use the setup skill).
---

# Troubleshoot connections

Find out where a connection between a client, the control plane, and
a workspace breaks, then fix it with the user's approval. For any
other symptom, use the troubleshoot skill.

## Source of truth

Read <https://coder.com/docs/llms.txt>, then the networking pages as
Markdown: `admin/networking.md`, `admin/networking/troubleshooting.md`,
`admin/networking/stun.md`, `admin/networking/port-forwarding.md`,
`admin/networking/wildcard-access-url.md`, and
`admin/templates/troubleshooting.md` under `https://coder.com/docs/`.
`coder <subcommand> --help` is authoritative for flags.

## User interaction

- Ask one question: what they ran or clicked, and what they saw.
- Explain in plain words. "Direct" means peer-to-peer. "Relayed" or
  "DERP" means traffic goes through the Coder server.
- Confirm before you change server flags, firewall rules, or the
  template.
- Never print session tokens or agent tokens. Redact them from logs.

## Preconditions

1. `coder whoami` succeeds. If not, ask the user to run
   `coder login <url>`.
2. If `CODER_AGENT_TOKEN` is set, you are inside a workspace. Tests
   from here measure this workspace, not the user's laptop.

## Triage, in order

Stop at the first layer that fails.

1. Agent state. Run `coder show <workspace>`. If the agent stays
   `connecting`, the agent or its init script failed. Go to
   "Agent stuck connecting".
2. Client to control plane. Run `coder netcheck`. It prints DERP and
   STUN information for this client.
3. Client to workspace. Run `coder ping <workspace> -n 5`. The output
   says whether pongs come direct or through a DERP relay, and lists
   client-side and agent-side reasons a direct connection failed.
   Add the global `-v` flag for client debug logs.
4. Throughput. Run `coder speedtest <workspace>`. Add `--direct` to
   wait for a direct connection first.
5. The specific tool: SSH, port forwarding, or apps (sections below).
6. Still unclear: `coder support bundle <workspace> --yes`. Tell the
   user where the file is and that it may contain sensitive data.

## Agent stuck connecting

- The workspace must reach the access URL. From inside the resource
  (for example `docker exec` or `kubectl exec`), run
  `curl -fsS "<access-url>/healthz"`. It must return `OK`.
- The image needs `curl`, `wget`, or `busybox` for the init script.
- Read `/tmp/coder-agent.log` and `/tmp/coder-startup-script.log`
  inside the resource.
- A reverse proxy in front of Coder must forward WebSockets.
- A template can set `url` in the `coder` provider block to reach the
  control plane by an internal address while you debug.

## Direct versus relayed

Direct connections need outbound UDP from both client and agent: to
the STUN servers on `udp/3478`, then on high ports to each other.
Browsers never connect direct. `coder ping` reports these causes:

| Cause | What to do |
| --- | --- |
| Direct disabled on the server | `--block-direct-connections` or `CODER_BLOCK_DIRECT` is set; leave it if intentional |
| UDP blocked | Allow UDP for clients, workspaces, and STUN servers |
| Hard NAT on both sides | See the STUN page; one side must have an easy NAT |
| No STUN servers | Check `--derp-server-stun-addresses` on the server |
| VPN as default route | Add a VPN exception or an internal STUN server |
| MTU under 1378 | Use another interface, or block direct and use DERP |

Relayed connections still work. They are slower when the user is far
from the server; workspace proxies help browser traffic.

## SSH

- Test with `coder ssh <workspace> -- true`. Use
  `<workspace>.<agent>` when the workspace has more than one agent.
- `coder ssh -l <dir>` writes SSH diagnostic logs to a directory.
- For plain `ssh`, IDEs, `scp`, and `rsync`, run
  `coder config-ssh --dry-run`, then `coder config-ssh -y`. Use
  `--no-wildcard` for tools that read hosts from `~/.ssh/config`.

## Port forwarding

- CLI: `coder port-forward <workspace> --tcp <local>:<remote>`. It
  runs until stopped, so run it in the background.
- Dashboard port forwarding needs a wildcard access URL. Each
  hostname label must stay under 63 characters, so long app, agent,
  workspace, and user names break it.
- Shared ports cannot exceed the template's maximum sharing level on
  Premium deployments.

## Apps do not load

1. Check that `CODER_WILDCARD_ACCESS_URL` is set and matches the
   wildcard domain. It must start with one `*`.
2. Resolve a test name: `dig test.<wildcard-domain>`.
3. Check that the TLS certificate covers the wildcard domain.
4. Check that the `coder_app` uses `subdomain = true` and a `url`
   the app really listens on.
5. Check the app's `healthcheck` block, if it has one.

## Fix and verify

1. State the fix in one sentence and get approval.
2. Apply it through the supervisor or template that owns it.
3. Re-run the failed triage step and `coder ping <workspace> -n 5`.
4. Hand off: what was wrong, what changed, how you verified it.

## Safeguards

- Do not disable TLS, auth, or WebSocket checks to make an error go
  away.
- Do not restart the server or rebuild workspaces without asking.
- Do not set `--block-direct-connections` without explaining that
  all traffic then goes through DERP.
