---
name: template-testing
description: >
  Test a Coder template change before users get it, then promote it.
  Use when the user asks to test, stage, dry-run, validate, or roll out
  a template update safely, push a version without activating it,
  build a test workspace from a specific template version, set up CI
  that validates and publishes templates, read workspace build logs
  for a new version, or promote, roll back, or archive template
  versions. Do not use for writing the template itself (use the
  templates skill), adding a module (use the modules skill), or
  contributing to coder/registry (use the registry-templates skill).
---

# Template testing

Push a template change as an inactive version, prove it builds a
working workspace, then promote it. Users keep the current active
version until you promote.

## Source of truth

Read <https://coder.com/docs/llms.txt>, then
`tutorials/testing-templates.md`,
`admin/templates/managing-templates/change-management.md`, and
`admin/templates/troubleshooting.md` under `https://coder.com/docs/`.
`coder templates push --help` and `coder create --help` are
authoritative for flags.

## User interaction

- Tell the user the plan in two lines: push inactive, build a test
  workspace, promote if it works.
- Ask once before you promote. Promotion changes what every new
  workspace gets.
- Do not ask for cloud credentials. Builds use the provisioner's
  credentials.

## Workflow

### 1. Validate locally

```sh
cd "$TEMPLATE_DIR"
terraform init
terraform fmt -check
terraform validate
```

If the repo already uses `tflint`, run it too. The Coder docs do not
cover `tflint`, so do not add it to a repo that does not use it.

### 2. Push without activating

```sh
VERSION="$(git rev-parse --short HEAD)"
coder templates push "$TEMPLATE_NAME" -d "$TEMPLATE_DIR" \
  --activate=false --name "$VERSION" \
  --message "<one-line change summary>" --yes
coder templates versions list "$TEMPLATE_NAME" -o json
```

Pass the same `--variable`, `--variables-file`, and
`--provisioner-tag` values the template uses today. Without
`--provisioner-tag`, the push reuses the active version's tags.

For a riskier change, push to a separate test template name instead
(for example `$TEMPLATE_NAME-test`), so nothing touches the real
template until you push there.

### 3. Build a test workspace from that version

```sh
coder create "test-$VERSION" \
  --template "$TEMPLATE_NAME" --template-version "$VERSION" \
  --parameter name=value --yes
```

- Supply every required parameter with `--parameter`, a preset with
  `--preset`, or `--rich-parameter-file <file.yaml>`.
- `--use-parameter-defaults` accepts defaults for anything not set.
- To test an upgrade of an existing workspace instead, run
  `coder update <workspace> --template-version "$VERSION" --yes`.

### 4. Check the result

1. `coder logs "test-$VERSION"` shows the build log. Read the first
   error, not the last.
2. `coder show "test-$VERSION"` must show the agent ready, not
   connecting.
3. Run a real command: `coder ssh "test-$VERSION" -- <build or test>`.
4. Open each app the change touched.

If the build fails, fix the template, push a new version, and repeat.

### 5. Clean up and promote

```sh
coder delete "test-$VERSION" --yes
coder templates versions promote \
  --template "$TEMPLATE_NAME" --template-version "$VERSION"
```

Delete the test workspace even when the test fails. Promote only
after the user says yes.

### 6. Roll back or tidy up

- Roll back by promoting the previous good version name from
  `coder templates versions list`.
- Archive failed versions with
  `coder templates versions archive "$TEMPLATE_NAME" <version> --yes`.

## CI

The testing-templates tutorial runs these same steps in GitHub
Actions with `coder/setup-action`. Use a dedicated machine account
token stored as `CODER_SESSION_TOKEN`, not a person's token. On
Premium, the docs recommend a service account.

## Safeguards

- Never push with the default `--activate=true` while testing.
- Never promote a version whose test workspace did not reach ready.
- Do not delete workspaces other than the test workspace you made.
- Do not put secrets in `--variable` values or in version messages.
