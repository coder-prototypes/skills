# Coder Skills

[![skills.sh](https://skills.sh/b/coder/skills)](https://skills.sh/coder/skills)

Skills for Claude Code, Codex, Cursor, and other coding agents to install, configure, and operate Coder.

## Admin edition

This `admin-skills` branch is the admin edition. It is the default
set plus skills for people who run Coder for a team, for users of a
deployment, and for contributors to the Coder Registry.

The `setup` skill asks new users if they want this edition before it starts.

```sh
# Claude Code
claude plugin marketplace add https://github.com/coder-prototypes/skills.git#admin-skills
claude plugin install coder-skills@coder-skills-admin

# Other agents
npx skills add https://github.com/coder-prototypes/skills/tree/admin-skills --global
```

## Catalog

### Admins

| Skill                      | Use it to                                                                  |
|----------------------------|----------------------------------------------------------------------------|
| `setup`                    | Install Coder, create the first admin, and build the first workspace.      |
| `production`               | Review HA, database, TLS, provisioners, and licenses before rollout.       |
| `upgrade`                  | Back up and upgrade Coder, then check that everything still works.         |
| `troubleshoot`             | Find and fix problems on a running deployment.                             |
| `troubleshoot-connections` | Fix agents stuck connecting, SSH, port forwarding, DERP, and apps.         |
| `access`                   | Set up GitHub or OIDC sign-in, users, groups, and roles.                   |
| `observability`            | Set up Prometheus metrics, logs, health checks, and alerts.                |
| `provisioners`             | Run external provisioner daemons with scoped keys and tags.                |
| `scale-testing`            | Load test Coder with `coder exp scaletest`, clean up, and read results.    |
| `templates`                | Write, edit, push, and version templates.                                  |
| `template-testing`         | Push an inactive version, test it in a workspace, then promote it.         |
| `mcp-templates`            | Create, test, and publish a template with only the Coder MCP tools.        |
| `support-bundle`           | Collect a support bundle for a workspace from a toolbox workspace.         |
| `modules`                  | Add or update registry modules in a template.                              |
| `coder-agents-admin`       | Enable Coder Agents: LLM providers, models, access, skills, and templates. |

### Users

| Skill           | Use it to                                                             |
|-----------------|-----------------------------------------------------------------------|
| `workspaces`    | Create, start, stop, and work inside workspaces from the CLI.         |
| `personalize`   | Set up dotfiles, `coder config-ssh`, VS Code, and JetBrains.          |
| `devcontainers` | Start, connect to, rebuild, and fix dev containers in a workspace.    |
| `tasks`         | Run coding agents in Coder. Coder Tasks was replaced by Coder Agents. |

### Registry contributors

| Skill                | Use it to                                                     |
|----------------------|---------------------------------------------------------------|
| `registry-templates` | Create or update a template in the coder/registry repository. |
| `registry-modules`   | Create or update a module in the coder/registry repository.   |

`registry-templates` and `registry-modules` are copied from
[`.agents/skills/coder-templates`](https://github.com/coder/registry/tree/main/.agents/skills/coder-templates)
and
[`.agents/skills/coder-modules`](https://github.com/coder/registry/tree/main/.agents/skills/coder-modules)
in [coder/registry](https://github.com/coder/registry) at commit
`6bc0014`, under the
[Apache-2.0 license](https://github.com/coder/registry/blob/main/LICENSE).
Changes: the frontmatter `name` and one em dash per file, replaced to
meet this repo's style lint.

## Usage

Install with the [skills CLI](https://skills.sh):

```sh
npx skills add coder/skills --global
```

Install with the [Claude Code plugin marketplace](https://claude.com/docs/plugins/marketplace):

```sh
claude plugin marketplace add coder/skills
claude plugin install coder-skills@coder-skills
```

After installing, ask your agent to install or set up Coder:

```text
╭────────────────────────────────────────────────────╮
│ >_ OpenAI Codex (v0.130.0)                         │
│                                                    │
│ model:     gpt-5.5 xhigh   fast   /model to change │
│ directory: ~/projects/coder/skills                 │
╰────────────────────────────────────────────────────╯

› setup coder

• I’ll use the setup skill because this is a first-time Coder deployment/bootstrap request. I’m going to read its workflow, then I’ll
  check the local repo context so I can choose the least surprising setup path.

• Explored
  └ Read SKILL.md (setup skill)
```

## License

[MIT](LICENSE), except `skills/registry-templates` and
`skills/registry-modules`, which are Apache-2.0 (see
[Registry contributors](#registry-contributors)).
