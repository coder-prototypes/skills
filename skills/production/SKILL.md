---
name: production
description: >
  Prepare and run a Coder deployment for production use. Use when the
  user asks to make Coder production-ready, review a deployment
  before rollout, set up high availability or more replicas, move to
  external PostgreSQL, plan backups, configure TLS, the access URL, or
  a wildcard access URL, move builds to external provisioners, size
  resources and quotas, check audit logs, or add or check a license.
  Ends with a production checklist. Do not use for first-time installs
  (use the setup skill), upgrades (use the upgrade skill), or
  debugging an outage (use the troubleshoot skill).
---

# Production

Review a running Coder deployment against the production
recommendations in the docs, fix gaps with the user's approval, and
leave a checklist.

## Source of truth

Read <https://coder.com/docs/llms.txt>, then as Markdown under
`https://coder.com/docs/`: `admin/setup.md`,
`admin/networking/high-availability.md`, `install/plan/sizing.md`,
`tutorials/best-practices/scale-coder.md`,
`install/operate/provisioners.md`, `install/prepare/licensing.md`, and
`admin/security/audit-logs.md`. `coder server --help` lists every
server flag and its environment variable.

## User interaction

- Ask how many users they expect and how Coder is installed (Helm,
  systemd, Docker compose).
- Report gaps as a short list: what is missing, why it matters, the
  fix.
- Get approval before each change. Most need a server restart.
- Never print database URLs with passwords, TLS keys, or license
  JWTs.

## Inspect

```sh
coder version
coder features list -o json
coder licenses list -o json
coder provisioner list -o json
curl -fsS "$CODER_URL/healthz"
```

Also read the deployment health page (`/health`) and the server
config from the supervisor (Helm values, systemd unit, compose file).

## Areas to review

### Access URL and TLS

- `CODER_ACCESS_URL` is an external `https://` URL, not localhost.
- Terminate TLS on the server (`CODER_TLS_ENABLE`, `--tls-cert-file`,
  `--tls-key-file`) or at a reverse proxy that forwards WebSockets.
- `CODER_REDIRECT_TO_ACCESS_URL=true` redirects plain HTTP.
- `CODER_WILDCARD_ACCESS_URL` (for example `*.coder.example.com`)
  needs matching DNS and a certificate that covers it. Do not use a
  top-level domain. After it works, consider `--disable-path-apps`.

### Database and backups

- Set `CODER_PG_CONNECTION_URL` to an external PostgreSQL (13 or
  newer). The docs recommend an external, managed database with
  daily backups for production instead of the bundled one.
- To migrate off the built-in database, follow the steps in
  `admin/setup.md`: stop Coder, `pg_dump` the built-in database,
  restore it into the external one, then start Coder with the new
  URL.
- Keep the database in the same region as the control plane, ideally
  under 10 ms away.
- Schedule backups and always back up before an upgrade. Downgrades
  need a database restore.

### High availability

- Coder enters HA mode when several replicas share one database. HA
  is a Premium feature. Run at least three replicas for API
  availability, in one region, spread across zones.
- Set `CODER_DERP_SERVER_RELAY_URL` and `CODER_CLUSTER_HOST` to each
  replica's own address. Allow TCP 6222 between replicas.
- With Helm, raise `coder.replicaCount`.
- Do not autoscale `coderd`. Size for weekly peak. Scale down and
  upgrade in a maintenance window.

### Provisioners

- Run external provisioners and set `CODER_PROVISIONER_DAEMONS=0`
  on the control plane. Use the provisioners skill.
- Each provisioner runs one build at a time. Size for peak
  concurrent builds.

### Resources and quotas

- Start from the validated architecture for the user count and
  watch real usage. The scale testing page suggests about 1 vCPU and
  2 GB per 250 users for each `coderd` replica.
- Set CPU and memory requests and limits for workspaces in the
  templates.
- Quotas (Premium) cap workspace spend with `daily_cost` in templates
  and allowances on groups.

### Audit logs and retention

- Audit logs need Premium and turn on with the license.
- Ship server logs (`--log-json`) to the log store; audit entries are
  in them.
- Set `--audit-logs-retention` to match compliance needs. The help
  text advises at least a year.

### License

- `coder licenses add -f <file.jwt>` adds a license. Have the user
  run it so the JWT does not enter chat.
- `coder licenses list` shows expiry. `coder features list` shows
  entitlements and seat use.

### Observability

Turn on metrics and logs with the observability skill.

## Production checklist

Hand this to the user with each item marked done or open:

1. HTTPS access URL; HTTP redirects to it.
2. Wildcard access URL with DNS and TLS, if apps need it.
3. External PostgreSQL, low latency, with tested backups.
4. Three or more replicas, relay URL and cluster host set (if HA).
5. External provisioners; built-in provisioners off.
6. Resource limits in templates; quotas if spend matters.
7. Valid license; audit log retention set.
8. Metrics scraped and alerts set; logs kept 30 to 90 days.
9. Sign-in through the company identity provider (use the access
   skill).
10. Upgrade and rollback plan written down (use the upgrade skill).

## Safeguards

- Do not restart replicas during working hours without asking.
- Every instance that connects to the database joins the HA
  cluster. Never point a test server at the production database.
- Do not delete the built-in database until the external one works.
