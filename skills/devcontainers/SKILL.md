---
name: devcontainers
description: >
  Use dev containers inside a Coder workspace. Use when the user asks
  to add or fix a `devcontainer.json`, start, rebuild, or connect to a
  dev container in their workspace, SSH, port-forward, or open VS Code
  into a dev container sub-agent, rename the container's agent, stop
  Coder from managing a container, or find out why a dev container
  will not start. Do not use for adding dev container support to a
  template (use the templates skill and the Coder docs on configuring
  a template for dev containers) or for general workspace commands
  (use the workspaces skill).
---

# Dev containers

Work with dev containers that run inside a Coder workspace. Each
running dev container shows up as a sub-agent with its own terminal,
apps, SSH, and port forwarding.

## Source of truth

Read <https://coder.com/docs/user-guides/devcontainers.md> and its
subpages `working-with-dev-containers.md`,
`customizing-dev-containers.md`, and
`troubleshooting-dev-containers.md`. The `devcontainer.json` format
is at <https://containers.dev/>. `coder <subcommand> --help` is
authoritative for flags.

## User interaction

- Ask which repository and which dev container they mean when there
  is more than one.
- Show a `devcontainer.json` change before you write it.
- Tell the user a rebuild recreates the container.

## Preconditions

- `coder whoami` succeeds and the workspace is running.
- The workspace is Linux, has Docker (Docker-in-Docker or a mounted
  socket), and has the `@devcontainers/cli`. Most templates with dev
  container support include both. If not, the template admin must add
  them.

## Workflow

### 1. Add or check `devcontainer.json`

Put it in `.devcontainer/devcontainer.json` (recommended),
`.devcontainer.json`, or `.devcontainer/<folder>/devcontainer.json`
for several configs in one repo. Minimal example:

```json
{
  "name": "My Dev Container",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu"
}
```

Coder settings go under `customizations.coder`:

- `"name": "<agent-name>"` sets the sub-agent name (lowercase
  letters, digits, hyphens).
- `"autoStart": true` starts it with the workspace, only when the
  template turns on discovery autostart.
- `"ignore": true` hides the container from Coder.

### 2. Start it

Coder discovers dev container configs in the workspace and shows them
on the workspace page with a **Start** button. If the template uses
`coder_devcontainer` or autostart, it starts by itself.

### 3. Find the agent name

```sh
coder show <workspace>
```

By default the name comes from the workspace folder, so
`/home/coder/my-app` becomes `my-app`.

### 4. Connect

| Task | Command |
| --- | --- |
| Run a command | `coder ssh <workspace>.<agent> -- <command>` |
| Open VS Code Desktop | `coder open vscode <workspace>.<agent>` |
| Forward a port | `coder port-forward <workspace>.<agent> --tcp 8080` |
| Plain SSH and IDEs | `coder config-ssh -y`, then the host from `coder config-ssh --dry-run` |

`coder ssh <workspace>` without the agent goes to the parent
workspace, not the container.

### 5. Rebuild after changes

Changes to `devcontainer.json` need a rebuild. Coder marks the
container **Outdated**; select **Rebuild** on the workspace page.

## Troubleshooting

1. Read `/tmp/coder-agent.log`, `/tmp/coder-startup-script.log`, and
   `/tmp/coder-script-*.log` in the parent workspace.
2. Check Docker: `docker ps`. With Docker-in-Docker, check that the
   daemon runs. With a mounted socket, check
   `/var/run/docker.sock` and that the user is in the `docker` group.
3. Check that `devcontainer.json` is valid JSON and the repo is
   cloned.
4. SSH fails: confirm the agent name with `coder show`, then
   `grep -i container /tmp/coder-agent.log`.
5. Slow start: use a prebuilt image and fewer features.

## Limits

- Linux workspaces only.
- `forwardPorts` with `host:port` for Docker Compose sidecars is not
  supported; use `coder port-forward` on the sub-agent.
- To expose a port through the parent agent, use `appPort` in
  `devcontainer.json`.

## Safeguards

- Do not rebuild a container while the user has unsaved work in it.
- Do not start or restart the Docker daemon without asking.
- Do not edit the template; hand that to the template admin.
