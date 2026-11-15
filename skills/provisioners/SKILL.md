---
name: provisioners
description: >
  Run and manage external Coder provisioner daemons. Use when the user
  asks to add, scale, or move provisioners, run builds outside the
  control plane, create, list, or delete provisioner keys, route
  templates to specific provisioners with tags, run provisioners on
  Kubernetes, Docker, or a VM, turn off built-in provisioners, or find
  why builds wait in "pending" because no provisioner picks them up.
  Do not use for template authoring (use the templates skill) or
  general outages (use the troubleshoot skill).
---

# Provisioners

Provisioner daemons run `terraform` for template imports and
workspace builds. Move them out of the control plane so builds run
near the infrastructure, with their own credentials, and scale on
their own.

## Source of truth

Read <https://coder.com/docs/install/operate/provisioners.md> and
<https://coder.com/docs/install/operate/provisioners/manage-provisioner-jobs.md>.
`coder provisioner <subcommand> --help` is authoritative for flags.

## User interaction

- Ask where workspaces run (cluster, cloud account, or host) and how
  many builds may run at the same time.
- Explain tags in one line: a template's tags pick which provisioners
  may build it.
- Never print a provisioner key. It shows once on create; have the
  user store it in a secret right away.

## Inspect

```sh
coder provisioner list -o json
coder provisioner keys list -o json
coder provisioner jobs list --status pending -o json
coder features list -o json
```

Starting organization-scoped provisioners needs Template Admin or
Owner. The scale testing docs describe external provisioners as a
Premium capability; check `coder features list` before you start.

## Workflow

### 1. Create a scoped key

```sh
coder provisioner keys create <key-name> --org <org>
# Or limit it to tagged jobs:
coder provisioner keys create <key-name> --org <org> \
  --tag environment=kubernetes
```

Keys are scoped to one organization and, optionally, a set of tags.
The docs recommend scoped keys over user tokens and over the global
pre-shared key (`--psk` is deprecated in the CLI).

### 2. Run the daemons

- Kubernetes: store the key in a secret, then install the
  `coder-v2/coder-provisioner` Helm chart with
  `provisionerDaemon.keySecretName`, `provisionerDaemon.keySecretKey`,
  `coder.env` `CODER_URL`, and `coder.replicaCount`. Run one daemon
  per pod.
- VM or Docker: set `CODER_URL` and `CODER_PROVISIONER_DAEMON_KEY`,
  then run `coder provisioner start`. On one VM, give each daemon its
  own `--cache-dir`.
- Name daemons with `--name` so they are easy to find in
  `coder provisioner list`.

Each daemon runs one build at a time. Run as many as the peak number
of builds that should run at once. Place them in the same cluster or
network as the workspaces they build.

### 3. Route templates with tags

A job with no tags runs only on a provisioner with no tags. A job
with tags runs only on a provisioner that has all of those tags; the
provisioner may have more.

```sh
coder templates push <template> --provisioner-tag environment=kubernetes
```

- `coder_workspace_tags` in Terraform can set tags per workspace, but
  they do not apply to the template import job. Pass
  `--provisioner-tag` on push too.
- If every provisioner is tagged, keep one untagged daemon or every
  untagged job stays pending.
- `--tag scope=user` makes a user-scoped provisioner that builds only
  for templates tagged `scope=user`.

### 4. Turn off built-in provisioners

After external daemons build successfully, set
`CODER_PROVISIONER_DAEMONS=0` (or `--provisioner-daemons=0`) on the
control plane and restart it with the user's approval.

### 5. Verify

1. `coder provisioner list` shows the new daemons with the expected
   tags and a recent "last seen" time.
2. Build one workspace from a tagged template and one from an
   untagged template; both leave `pending`.
3. On Kubernetes, the pod logs show
   `provisioner: successfully connected to coderd`.

## Stuck or failed jobs

- `coder provisioner jobs list --status pending` shows queued jobs
  and their tags. Compare them with `coder provisioner list` tags.
- Cancel a stuck job with `coder provisioner jobs cancel <job-id>`
  after the user agrees.
- Stop daemons with `SIGHUP` to let the current build finish. `SIGINT`
  cancels it and fails the build.

## Safeguards

- Do not delete a key with `coder provisioner keys delete` while
  daemons still use it.
- Do not set `CODER_PROVISIONER_DAEMONS=0` before an external daemon
  works.
- Do not store keys in templates, Helm values in Git, or chat.
