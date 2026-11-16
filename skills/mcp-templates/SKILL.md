---
name: mcp-templates
description: >
  Create, test, and publish a Coder template using only the Coder MCP
  tools (coder_*), with no CLI or Git checkout. Use when the user asks
  to make a first template, add a template for their infrastructure,
  or change an existing template from a Coder Agents chat. For writing
  Terraform in a repository and pushing with the CLI, use templates.
---

# Templates with the Coder MCP tools

The Coder MCP tools act as the signed-in user, so every step below runs
with that user's permissions. Template authoring needs the Template
Admin or Owner role.

## 1. Check what exists

1. Call `coder_get_authenticated_user` and `coder_list_organizations` to
   learn who you are and which organization to use.
2. Call `coder_list_templates`. If a template already fits the user's
   infrastructure, offer to test or adapt it before you write a new one.
3. Ask where Coder runs (Docker, Kubernetes, a VM on a cloud) if you do
   not know yet. The infrastructure decides the Terraform provider.

## 2. Write the files

Start from the matching starter in
[coder/registry](https://registry.coder.com/templates) and keep it
small: one `coder_agent`, the compute resource, and at most two modules
(for example code-server and Git clone). A template needs:

- `terraform { required_providers { coder = { source = "coder/coder" } } }`
  plus the infrastructure provider.
- `data "coder_workspace" "me" {}` and `data "coder_workspace_owner" "me" {}`.
- One `coder_agent` with `arch` and `os` matching the compute.
- A compute resource that runs `coder_agent.<name>.init_script` with
  `CODER_AGENT_TOKEN` set to `coder_agent.<name>.token`, and that scales
  to zero with `data.coder_workspace.me.start_count`.

For Docker, replace `localhost` and `127.0.0.1` in the init script with
`host.docker.internal` and add `host { host = "host.docker.internal"
ip = "host-gateway" }`, or the agent cannot reach Coder.

## 3. Upload and import

1. Call `coder_upload_tar_file` with `files` as a map of file name to
   contents, for example `{"main.tf": "...", "README.md": "..."}`. Keep
   the result's file ID.
2. Call `coder_create_template_version` with that `file_id` and the
   `organization_id`. Pass `template_id` only when you update an
   existing template.
3. Call `coder_get_template_version_logs` with the new version ID until
   the import finishes. On failure, read the Terraform error, fix the
   files, and repeat from step 1. Do not guess at fixes; quote the error
   line to the user when it is not obvious.

## 4. Publish

- New template: call `coder_create_template` with `name`, a
  `display_name`, a one-line `description`, an `icon` such as
  `/icon/docker.png`, and the `version_id`.
- Existing template: confirm with the user, then call
  `coder_update_template_active_version` with `template_id` and
  `template_version_id`.

## 5. Test it

1. Call `coder_template_version_parameters` and pick values with the
   user.
2. Call `coder_create_workspace` with the template, a name such as
   `<template>-test`, and the parameters.
3. Call `coder_get_workspace` until the build finishes, then
   `coder_get_workspace_build_logs` if it failed and
   `coder_get_workspace_agent_logs` if the agent did not connect.
4. When it works, tell the user the template name and the workspace URL,
   and offer to stop or delete the test workspace with
   `coder_create_workspace_build` (`transition` = `stop` or `delete`).

## Rules

- Never put secrets in template files. Use parameters with
  `ephemeral = true`, or external auth, for credentials.
- Confirm before you change the active version of a template people
  already use.
- Deleting a template is not available through these tools; send the
  user to the dashboard.
