---
name: upgrade
description: >
  Upgrade an existing Coder deployment to a newer release safely. Use
  when the user asks to upgrade, update, or patch Coder, move between
  stable and mainline, or check if their version is out of date. Covers
  backups, release notes, the upgrade itself for systemd, Docker
  compose, Helm, and standalone installs, and post-upgrade checks. Do
  not use for first-time installs (use the setup skill) or for
  debugging unrelated failures (use the troubleshoot skill).
---

# Upgrade

Move a running Coder deployment to a newer release with a backup and
a way back.

## Source of truth

Read <https://coder.com/docs/llms.txt>, then the upgrade page and the
release notes for every version between the current and target
release. Breaking changes and required steps live there, not here.

## User interaction

- Tell the user the current version, the target, and the notable
  changes in two or three plain sentences.
- Ask for one yes/no before you start. Mention the expected downtime.
- Default to the latest stable release unless the user asks for
  mainline.

## Workflow

### 1. Inspect

```sh
coder version
curl -fsS "$CODER_URL/api/v2/buildinfo"
```

Find the supervisor (systemd, Docker compose, Helm, or standalone)
and the database (built-in or external PostgreSQL).

### 2. Back up

- External PostgreSQL: `pg_dump` to a file the user names, or confirm
  their managed backup ran today.
- Built-in database: stop the server, then copy the data directory.
- Helm: save current values with `helm get values`.

Do not continue without a backup or the user's explicit choice to
skip it.

### 3. Upgrade

Use the method that installed Coder:

- Standalone or package: `curl -fsSL https://coder.com/install.sh | sh`
  with `--stable` or `--version`, then restart the service.
- Docker compose: change the image tag, then `docker compose pull`
  and `docker compose up -d`.
- Helm: `helm repo update`, then `helm upgrade` with the saved values
  and `--version`.

Migrations run on start. Do not interrupt the first start.

### 4. Verify

1. `/healthz` returns OK and `buildinfo` shows the new version.
2. Deployment health has no new errors.
3. Provisioners reconnected.
4. One existing workspace starts and its agent reaches ready.
5. Workspace agents update on their next start; tell the user.

### 5. Roll back if needed

Restore the backup and the previous version. Migrations do not run
backwards, so a rollback always needs the database backup.

## Safeguards

- Never skip more major versions than the release notes allow.
- Never run two Coder versions against one database.
- Do not delete the backup until the user confirms things work.
