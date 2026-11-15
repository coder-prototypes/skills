---
name: scale-testing
description: >
  Plan, run, clean up, and read the results of a Coder load test with
  `coder exp scaletest`. Use when the user asks to load test or scale
  test Coder, check how many users or workspaces a deployment can
  handle, simulate SSH, web terminal, app, or dashboard traffic,
  validate sizing before a rollout, or remove leftover `scaletest-`
  users and workspaces. Do not use for general production review (use
  the production skill), ongoing monitoring (use the observability
  skill), or debugging a live outage (use the troubleshoot skill).
---

# Scale testing

Generate realistic load against a Coder deployment, watch how it
behaves, clean up, and explain what the results mean.

## Source of truth

Read <https://coder.com/docs/install/validate/scale-testing.md> for
the method and
<https://coder.com/docs/install/validate/scale-testing/utility.md>
for the CLI. `coder exp scaletest <subcommand> --help` is
authoritative. The `exp` commands are experimental and change between
releases, so read the help every session.

## User interaction

- Warn first: the utility is experimental, can disrupt users, and can
  leave resources behind. The docs recommend a staging deployment or
  a dedicated cluster. Get a clear yes before you touch production.
- Agree on the target in one sentence, for example "500 workspaces,
  SSH traffic for 30 minutes".
- Report results in plain words: what held up, what broke, and what
  to change.

## Plan the load

1. Pick the scenarios. The docs test, in overlapping stages:
   workspace creation, SSH, web terminal, workspace apps, and the
   dashboard.
2. Pick counts from the expected peak, not the total user count.
   Inactive users use no resources.
3. Use a small test template. The `scaletest-runner` template in
   `coder/coder` under `scaletest/templates` is built for this.
4. Make sure there are enough provisioners. Each runs one build at a
   time, so creation concurrency above the provisioner count only
   queues.
5. Turn on metrics first (observability skill), including
   `CODER_PROMETHEUS_COLLECT_DB_METRICS=true`, and have server logs
   in a searchable store.
6. `create-workspaces` and `dynamic-parameters` say to turn off rate
   limits on the server first, because they log in many users from
   one IP.

## Subcommands

`coder exp scaletest --help` lists: `create-workspaces`,
`workspace-traffic`, `dashboard`, `cleanup`, `create-users`,
`autostart`, `dynamic-parameters`, `notifications`, `prebuilds`,
`workspace-updates`, `chat`, `bridge`, `agentfake`, `llm-mock`, and
`smtp`. The docs cover the first four.

## Run

Create workspaces and keep them for later traffic tests:

```sh
coder exp scaletest create-workspaces \
  --template "$TEMPLATE" --count 100 --concurrency 10 \
  --retry 5 --timeout 2h --job-timeout 2h \
  --no-cleanup \
  --output json:"$RESULTS_DIR/create-workspaces.json"
```

Generate traffic against them (SSH here; omit `--ssh` for web
terminal traffic, or use `--app` for workspace app traffic):

```sh
coder exp scaletest workspace-traffic \
  --template "$TEMPLATE" --ssh \
  --bytes-per-tick 1024 --tick-interval 100ms \
  --timeout 30m --job-timeout 30m \
  --target-workspaces 0:100 \
  --output json:"$RESULTS_DIR/traffic-ssh.json"
```

Load the dashboard API:

```sh
coder exp scaletest dashboard --timeout 30m \
  --output json:"$RESULTS_DIR/dashboard.json"
```

Run long tests in the background and check progress; they run until
`--timeout`.

## Clean up

Always clean up, including after a failed or interrupted run:

```sh
coder exp scaletest cleanup \
  --cleanup-timeout 30m --cleanup-job-timeout 2h
```

This deletes workspaces and users with the `scaletest-` prefix. Then
run `coder list --all -o json` and `coder users list -o json` to
confirm none are left. Check the cloud account for orphaned
resources.

## Read the results

- The docs call a test good when users saw no interruptions, `coderd`
  did not crash or restart, and no other internal errors appeared.
- Read the JSON output files for failed runs and their errors.
- API latency: workspace, template, and build calls should answer
  within about 300 ms. Watch `coderd_api_request_latencies_seconds`.
- Database: high `coderd_db_query_latencies_seconds` tail latency
  means the database needs more resources.
- Builds: long queue times mean more provisioners.
- Error logs during the test point to an under-sized deployment.
- Compare against the validated architectures in
  `install/plan/sizing.md`. The scale test numbers in the docs are
  for reference, not sizing rules.

## Safeguards

- Never run against production without explicit approval.
- Never skip cleanup. Do not delete users or workspaces that lack the
  `scaletest-` prefix.
- Do not leave server rate limits off after the test.
