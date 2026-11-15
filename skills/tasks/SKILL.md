---
name: tasks
description: >
  Run coding agents against Coder workspaces as a user. Use when the
  user mentions Coder Tasks, wants to hand work to an AI agent that
  runs in Coder, start a background agent from a prompt, script or
  automate agent runs, run Claude Code or Codex inside a workspace, or
  move an integration off the removed Tasks API. Explains that Coder
  Tasks was removed and routes to Coder Agents chats or an agent
  module in a workspace. Do not use for enabling Coder Agents or
  adding LLM providers (use the coder-agents-admin skill) or for plain
  workspace commands (use the workspaces skill).
---

# Tasks

Help a user run coding agents in Coder. Coder Tasks was deprecated in
v2.34 and has been removed, including the Tasks API
(`/api/v2/tasks`), and the current CLI has no `task` command. This
skill routes the user to the supported paths instead.

## Source of truth

Read <https://coder.com/docs/ai-coder.md> for the options,
<https://coder.com/docs/ai-coder/agents.md> for Coder Agents, and
<https://coder.com/docs/ai-coder/agents/tasks-to-chats-migration.md>
for moving off the Tasks API. If the deployment runs a release older
than v2.36, read the docs for that release about Tasks; this skill
covers current releases only.

## User interaction

- Say once, in plain words: "Coder Tasks was replaced by Coder
  Agents." Then help with what they wanted to do.
- Ask one question: do they want a background agent from a prompt, or
  an agent they pair with inside a workspace?
- Never put session tokens or LLM keys in chat, prompts, or links.

## Pick a path

| Want | Use |
| --- | --- |
| Background or parallel work from a prompt | Coder Agents chats |
| Automation from CI, webhooks, or scripts | Coder Agents Chats API |
| Pair with Claude Code or Codex in a terminal | An agent module in the workspace template |
| Agent inside the editor | IDE agents (Cursor, Copilot, and similar) |

## Coder Agents chats

1. Check it is on: open `<access-url>/agents`. If there is no
   Agents page or no model, an admin must set it up (coder-agents-admin
   skill).
2. Pick a model, write the prompt, and send it. The agent runs in the
   control plane and creates a workspace from a matching template
   when it needs files, commands, or code changes.
3. Group related chats in a project to share instructions, skills,
   and memory (early access).
4. Share a ready-made prompt as a link:
   `<access-url>/agents?prompt=<url-encoded text>`. The user still
   selects **Send**. Never put secrets in the link.

Give the agent context from the workspace:

- `AGENTS.md` in `~/.coder/` or the agent's working directory is
  added to every chat that uses that workspace.
- Skills in `.agents/skills/<name>/SKILL.md` (also `skills/`,
  `.claude/skills/`, `.codex/skills/`) are found automatically.
- Inside the workspace, `coder exp chat context list` shows extra
  scan roots, `coder exp chat context add <path>` adds one, and
  `coder exp chat context refresh` re-pins chats to the latest
  context. These are experimental commands.

## Chats API

For automation, create a chat with `POST /api/v2/chats` and a
`content` array of text parts, then stream updates from
`GET /api/v2/chats/{chat}/stream` (WebSocket). Use a scoped API token
from `coder tokens create` kept in an environment variable or CI
secret, not a person's session token on the command line.

Mapping from the removed Tasks API, from the migration page:

| Tasks API | Chats API |
| --- | --- |
| `POST /api/v2/tasks/{user}` | `POST /api/v2/chats` |
| `POST .../tasks/{user}/{task}/send` | `POST /api/v2/chats/{chat}/messages` |
| `GET .../tasks/{user}/{task}/logs` | `GET /api/v2/chats/{chat}/stream` |
| `POST .../tasks/{user}/{task}/pause` | `POST /api/v2/chats/{chat}/interrupt` |
| `DELETE /api/v2/tasks/{user}/{task}` | `PATCH /api/v2/chats/{chat}` with `{"archived": true}` |

Templates no longer need `coder_ai_task` or `coder_task`; any
template with a clear description works.

## Agent in a workspace

A template admin adds a terminal agent with a registry module, for
example `registry.coder.com/coder/claude-code/coder`. The user then
creates a workspace from that template (workspaces skill) and works
with the agent over `coder ssh` or their IDE. Here the agent runs in
the workspace, so read the module README for how it gets LLM
credentials. Coder Agents keeps credentials in the control plane.

## Safeguards

- Do not create many chats or workspaces in a loop without the
  user's go-ahead; each turn costs LLM tokens.
- Do not call `/api/v2/tasks`; it no longer exists.
- Review agent changes before merging; the docs say the agent does
  not replace developer review.
