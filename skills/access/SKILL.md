---
name: access
description: >
  Manage who can sign in to a Coder deployment and what they can do.
  Use when an admin asks to add or remove users, set up GitHub or
  OIDC sign-in (Okta, Entra ID, Google, Keycloak), sync groups or
  roles from the identity provider, create groups, assign roles, or
  audit access. Do not use for first-time installs (use the setup
  skill) or for workspace-level sharing between users (use the
  workspaces skill).
---

# Access

Set up sign-in and permissions on a running Coder deployment.

## Source of truth

Read <https://coder.com/docs/llms.txt>, then the users, OIDC, groups
and roles, and IdP sync pages for the user's provider. Settings names
and Premium features change between releases.

## User interaction

- Ask which identity provider they use and whether everyone should
  sign in through it.
- Translate terms: "identity provider" is the company login, a
  "group" is a set of people, a "role" is what they may do.
- Never ask the user to paste client secrets into chat. Have them
  export the secret in the shell and read it from the environment.

## Workflow

### 1. Inspect

```sh
coder whoami
coder users list -o json
coder groups list -o json
curl -fsS "$CODER_URL/api/v2/users/authmethods"
```

Confirm the user is an Owner or User Admin.

### 2. Sign-in method

- GitHub: create an OAuth app with the callback from the docs, set
  the client ID and secret on the server, and restrict to the user's
  org.
- OIDC: register Coder with the provider, set issuer URL, client ID,
  secret, and the email domain. Test with one account before telling
  everyone.
- Turn off password login only after an Owner signs in with the new
  method.

Apply config through the supervisor that runs Coder, then restart
with the user's approval.

### 3. Groups and roles

- Create groups that match teams: `coder groups create <name>`.
- Map provider groups or roles with IdP sync when the user has
  Premium; otherwise add members by hand.
- Keep the Owner role to two or three people.

### 4. Verify

1. A test user signs in with the new method.
2. The user lands in the expected groups and roles.
3. The audit log shows the changes.

## Safeguards

- Never lock out the last Owner. Keep one password or break-glass
  account until the new method works.
- Do not delete users with workspaces; suspend them instead.
- Do not print client secrets or tokens.
