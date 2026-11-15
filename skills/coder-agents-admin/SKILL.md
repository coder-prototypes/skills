---
name: coder-agents-admin
description: >
  Enable and administer Coder Agents, the chat agent that runs in the
  Coder control plane, for a deployment or an organization. Use when
  an admin asks to turn on Coder Agents, add an LLM provider (Anthropic,
  OpenAI, Google, Azure OpenAI, AWS Bedrock, OpenRouter, Vercel, or an
  OpenAI-compatible endpoint), add models or set a default model,
  control who can use agents, write system or organization
  instructions, publish organization skills, explain projects, choose
  which templates agents may use, or plan concurrency and spend. Do
  not use for running coding agents as a user (use the tasks skill)
  or first-time installs (use the setup skill).
---

# Coder Agents admin

Set up Coder Agents for an organization: a provider, at least one
model, access, instructions, skills, and templates the agent can
pick. The agent loop runs in the control plane, so LLM keys never
enter workspaces.

## Source of truth

Read <https://coder.com/docs/llms.txt>, then as Markdown under
`https://coder.com/docs/ai-coder/agents/`: `getting-started.md`,
`models.md`, `platform-controls.md`,
`platform-controls/organizations.md`,
`platform-controls/template-optimization.md`,
`extending-agents.md`, and `projects.md`. Menu names and APIs change
between releases; trust the docs over this file.

## User interaction

- Ask which provider they have a key for and which organization
  should use agents.
- Never ask for the API key in chat. The admin enters it in the
  dashboard, or follows the `read -s` handoff in the setup skill's
  `references/coder-agents.md`.
- Recommend a pilot of three to five people before a wide rollout.

## Preconditions

- Coder on the latest release. `coder version` and `coder whoami`.
- Providers need a deployment admin. Models need Owner or
  Organization Admin in that organization.
- The control plane can reach the LLM provider. Workspaces do not
  need to.

## Workflow

### 1. Provider (deployment-wide)

**Admin settings** > **AI** > **Providers** > **Add provider**. Pick
the type, a unique lowercase name, the credentials, and the endpoint.
Keys are stored encrypted and never shown again.

### 2. Models (per organization)

**Admin settings** > **AI** > **Models**, pick the organization, then
**Add model** with the provider's model identifier, a display name,
and the context limit. The first model becomes the default. Repeat in
every organization that uses agents. Start with one frontier model.

### 3. Access

- Every organization member gets the **Coder Agents User**
  (`agents-access`) role through the default member roles.
- To limit agents to some members (Premium), remove it from the
  default roles, for example
  `coder organizations edit --org <org> --default-org-member-roles organization-workspace-access`
  (keep any other default roles), then grant `agents-access` by IdP
  role sync or on **Admin settings** > **Organizations** >
  **Members**.
- Users also need read access to a model. New models are shared with
  the whole organization.
- Service accounts do not inherit `agents-access`; assign it
  directly.

### 4. Instructions

**Admin settings** > **AI** > **Coder Agents** > **Instructions**
holds the deployment system prompt and per-organization
instructions. Put coding standards, commit format, and branch rules
there.

### 5. Templates

- The agent picks a template from its name, description, and the top
  of its README. It does not read Terraform. Write specific
  descriptions: stack, repo, and kind of work.
- Block a template from agents with
  `coder templates edit --agents-allowed=false <template>` or on
  **Admin settings** > **AI** > **Coder Agents** > **Templates**.
- Consider dedicated agent templates with tighter network egress, and
  prebuilt workspaces for fast starts.

### 6. Skills and projects

- Organization skills live under **Admin settings** > **AI** >
  **Coder Agents** > **Skills**. Each is one `SKILL.md` up to 64 KB,
  with no supporting files, up to 100 per organization. A new skill is
  shared with **Everyone**; narrow it with **Manage permissions**.
- Workspace skills: templates and repos can ship skills in
  `.agents/skills/<name>/SKILL.md`; the agent finds them when it
  attaches to the workspace.
- Projects (early access) are created by users. Each groups chats with
  shared instructions, skills, and memory, and belongs to one
  organization.

### 7. Limits and spend

- Community and Premium licenses run up to 5 agents at once; more
  queue. AI Premium removes the cap until its Agent Hours hard limit.
- AI Gateway budgets per group cap spend and need a license that
  includes AI Gateway.

### 8. Verify

Open `<access-url>/agents`, pick the model, and send a short prompt
that needs a workspace. Confirm it picks the right template and the
workspace reaches ready.

## Safeguards

- Do not print or log provider keys.
- Do not remove `agents-access` from default roles without telling
  the admin that members lose access until granted again.
- Do not share an organization skill wider than the admin asked.
