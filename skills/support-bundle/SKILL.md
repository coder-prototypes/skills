---
name: support-bundle
description: >
  Collect a Coder support bundle for a workspace from a Coder Agents
  chat. Use when the user asks for a support bundle, wants to send
  diagnostics to Coder support, or a workspace connection problem needs
  deployment, agent, and network details in one file. Sets up a small
  admin toolbox workspace with the Coder CLI first if none exists.
---

# Support bundles

`coder support bundle` collects deployment health, the workspace and its
build logs, agent logs, network checks (DERP, STUN, `coder netcheck`),
and optional pprof data into one zip. It runs from the Coder CLI, so the
chat needs a workspace that has the CLI installed and signed in as the
user. The Coder MCP tools cannot produce a bundle on their own.

## 1. Find the workspace to diagnose

Call `coder_list_workspaces` (or ask the user) and confirm the target as
`<owner>/<workspace>`, plus the agent name when it has more than one.

## 2. Get a toolbox workspace

Call `coder_list_templates` and look for `admin-toolbox`. If it does not
exist, create it with the steps in the mcp-templates skill and these
files. This version is for Docker; for other infrastructure, keep the
`coder_agent` block and swap the compute resource.

`main.tf`:

```hcl
terraform {
  required_providers {
    coder  = { source = "coder/coder" }
    docker = { source = "kreuzwerker/docker" }
  }
}

data "coder_workspace" "me" {}
data "coder_workspace_owner" "me" {}

locals {
  # Inside the container, localhost is the container itself.
  access_url = replace(data.coder_workspace.me.access_url, "/localhost|127\\.0\\.0\\.1/", "host.docker.internal")
}

resource "coder_agent" "main" {
  arch = "amd64"
  os   = "linux"
  env = {
    CODER_URL           = local.access_url
    CODER_SESSION_TOKEN = data.coder_workspace_owner.me.session_token
  }
  startup_script = <<-EOT
    set -e
    mkdir -p "$HOME/.local/bin"
    curl -fsSL "$CODER_URL/bin/coder-linux-amd64" -o "$HOME/.local/bin/coder"
    chmod +x "$HOME/.local/bin/coder"
  EOT
}

resource "docker_container" "workspace" {
  count      = data.coder_workspace.me.start_count
  image      = "codercom/example-base:ubuntu"
  name       = "coder-${data.coder_workspace_owner.me.name}-${lower(data.coder_workspace.me.name)}"
  hostname   = data.coder_workspace.me.name
  entrypoint = ["sh", "-c", replace(coder_agent.main.init_script, "/localhost|127\\.0\\.0\\.1/", "host.docker.internal")]
  env        = ["CODER_AGENT_TOKEN=${coder_agent.main.token}"]
  host {
    host = "host.docker.internal"
    ip   = "host-gateway"
  }
}
```

The session token comes from `coder_workspace_owner` and belongs to
whoever owns the toolbox workspace, so the CLI has exactly that user's
permissions. Only admins should own toolbox workspaces, because a bundle
includes deployment details.

Then start a workspace from it with the chat's own `create_workspace`
tool, not `coder_create_workspace`: the chat can only run commands in a
workspace attached to it. Reuse an existing `admin-toolbox` workspace
the user owns when there is one, and wait for its agent to connect.

## 3. Collect the bundle

In the toolbox workspace, run:

```sh
export PATH="$HOME/.local/bin:$PATH"
coder support bundle <owner>/<workspace> [agent] --yes \
  --output-file /tmp/coder-support-<workspace>.zip
```

- Add `--template <template>` to include the template.
- Add `--pprof` only when the user reports high CPU or memory.
- Add `--workspace-file <path>` (repeatable) for logs inside the target
  workspace, for example `--workspace-file ~/.coder-agent.log`.

Check that the zip exists and is not empty, then attach it to the chat
so the user can download it.

## 4. Hand off

Tell the user what the bundle contains and that it may include internal
hostnames, IPs, and logs, so they should share it only with Coder
support. Offer to stop the toolbox workspace when done.

## Rules

- Never print `CODER_SESSION_TOKEN` or paste it into the chat.
- Do not collect bundles for workspaces the user did not name.
- If `coder support bundle` fails with 403, the toolbox owner lacks
  access to the target; ask an Owner to run it.
