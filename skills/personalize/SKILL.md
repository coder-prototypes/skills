---
name: personalize
description: >
  Personalize how a user works with their Coder workspaces: apply a
  dotfiles repo, use a `~/personalize` script, set up SSH with `coder
  config-ssh`, connect VS Code, Cursor, or JetBrains IDEs, and add the
  Coder Git SSH key to a Git host. Use when the user asks to bring
  their shell config, editor settings, or tools into a workspace, to
  connect a desktop IDE, to make `ssh`, `scp`, or `rsync` work with a
  workspace, or to fix their IDE connection. Do not use for creating
  or deleting workspaces (use the workspaces skill) or for changing a
  template (use the templates or modules skill).
---

# Personalize

Make Coder workspaces feel like the user's own machine: their
dotfiles, their SSH setup, and their IDE.

## Source of truth

Read as Markdown under `https://coder.com/docs/user-guides/`:
`workspace-dotfiles.md`, `workspace-access.md`,
`workspace-access/vscode.md`, `workspace-access/jetbrains.md` and its
`toolbox.md` and `gateway.md` subpages, and
`workspace-access/cursor.md`. `coder <subcommand> --help` is
authoritative for flags.

## User interaction

- Ask what they want: dotfiles, SSH, or which IDE.
- Never ask for a session token in chat. IDE sign-in flows open a
  browser or read the CLI's login.
- Show changes to `~/.ssh/config` with a dry run before you write.

## Preconditions

- `coder whoami` succeeds on the user's machine. If not, ask them to
  run `coder login <url>`.
- If `CODER_AGENT_TOKEN` is set, you are inside a workspace. Dotfiles
  run here; SSH config and IDE setup belong on the user's laptop.

## Dotfiles

- If the template asks for a dotfiles URL when the workspace is
  created (the `dotfiles` module), set it there; it applies on every
  start.
- Otherwise, inside the workspace run:

  ```sh
  coder dotfiles --yes <git-repo-url>
  ```

  Add `-b <branch>` for a branch. Coder clones the repo, then runs the
  first of `install.sh`, `install`, `bootstrap.sh`, `bootstrap`,
  `script/bootstrap`, `setup.sh`, `setup`, or `script/setup`.
  Without a script, it symlinks the dotfiles into `$HOME`
  (`--symlink-dir` changes the target).
- The script must be executable. If it is not, the user runs
  `chmod +x <script>` in the dotfiles repo, commits, and pushes.
- Some templates run `~/personalize` on start (the `personalize`
  module). Put per-workspace setup commands there.

## SSH from the user's machine

```sh
coder config-ssh --dry-run
coder config-ssh -y
```

- This adds a host block to `~/.ssh/config`. The dry run prints the
  host names to use, for example `coder.<workspace>`.
- `--no-wildcard` writes one entry per workspace and agent, for tools
  that list hosts from `~/.ssh/config`.
- `-o ForwardAgent=yes` adds an SSH option to every host.
- After this, `ssh`, `scp`, `rsync`, and IDE remote extensions work
  against the workspace.

## Git over SSH

`coder publickey` prints the user's Coder SSH public key for Git.
The user adds it to their Git host. `--reset` makes a new key and
breaks the old one everywhere; ask first.

## IDEs

### VS Code and Cursor

- Select **VS Code Desktop** on the workspace page, or run
  `coder open vscode <workspace> [<directory>]`. This installs the
  Coder Remote extension, signs in, and connects.
- Manual install: `ext install coder.coder-remote` in VS Code Quick
  Open.
- Cursor uses the same Coder extension.
- For connection problems, the extension's **Coder: Create Support
  Bundle** command collects logs.

### JetBrains

- Supported through JetBrains Toolbox and Gateway, for IntelliJ
  IDEA, PyCharm, GoLand, WebStorm, and the other IDEs listed in the
  docs.
- Toolbox: install the **Coder** provider from **Manage Providers**,
  enter the deployment URL, sign in, and pick the workspace.
- Internal CA: set **CA Path** in the Coder provider settings.

### Browser IDEs

code-server, VS Code Web, and other web IDEs show up as apps on the
workspace page when the template includes them.

## Verify

1. `ssh coder.<workspace> -- true` (or the host from the dry run)
   exits 0.
2. The IDE opens the workspace folder.
3. A dotfiles change, such as a shell alias, works in a new terminal.

## Safeguards

- Do not edit `~/.ssh/config` by hand; let `coder config-ssh` manage
  its block.
- Do not run `coder publickey --reset` without asking.
- Do not put tokens in dotfiles repos or JetBrains URIs you share.
