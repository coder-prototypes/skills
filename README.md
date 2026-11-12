# Coder Skills

[![skills.sh](https://skills.sh/b/coder/skills)](https://skills.sh/coder/skills)

Skills for Claude Code, Codex, Cursor, and other coding agents to install, configure, and operate Coder.

## Admin edition

This `admin-skills` branch is the admin edition. It adds three skills for people who run Coder for a team:

| Skill          | Use it to                                                          |
|----------------|--------------------------------------------------------------------|
| `troubleshoot` | Find and fix problems on a running deployment.                     |
| `upgrade`      | Back up and upgrade Coder, then check that everything still works. |
| `access`       | Set up GitHub or OIDC sign-in, users, groups, and roles.           |

The `setup` skill asks new users if they want this edition before it starts.

```sh
# Claude Code
claude plugin marketplace add https://github.com/coder-prototypes/skills.git#admin-skills
claude plugin install coder-skills@coder-skills-admin

# Other agents
npx skills add https://github.com/coder-prototypes/skills/tree/admin-skills --global
```

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

[MIT](LICENSE)
