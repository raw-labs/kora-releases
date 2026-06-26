---
name: kora-cli
description: >
  Read when installing the Kora CLI (@kora-platform/cli), signing up or
  logging in from a terminal, pointing the CLI at a Kora deployment, creating
  or selecting an organization, or wiring a coding agent or CI job to run
  kora commands non-interactively.
---

# Kora CLI

`kora` is the command-line client for a reachable Kora Platform deployment.
Use this skill when the user has a Kora base URL, was given access to an
existing deployment, or wants to manage Kora resources from a terminal.
Installing or operating the deployment itself is the `kora-self-host` skill,
not this one.

## Install

```sh
npm install -g @kora-platform/cli
kora --help
```

Requires Node.js >=24.0.0.

## Point The CLI At A Deployment

Base URL resolution order, first match wins:

1. `--base-url` flag on `kora auth login` / `kora auth signup`
2. `KORA_BASE_URL` environment variable
3. nearest `.kora/cli.toml` walking up from the working directory
4. `~/.config/kora/config.toml` (respects `XDG_CONFIG_HOME`)
5. interactive prompt during login/signup, persisted to the global config

After login the base URL is stored in the session, so subsequent commands
need no flag. Self-managed local-mode installs serve
`http://localhost:3000`.

## Authentication

`kora auth login` has two flows:

- **Device approval (the agent path).** In a non-interactive shell — including
  a coding agent's shell tool — `kora auth login` automatically switches to
  the browser device-approval flow; `--device` forces it from any terminal.
  The command prints a verification URL and a confirmation code, then waits:

  ```sh
  kora auth login --base-url http://localhost:3000 --device
  ```

  The user opens `<base-url>/device?code=XXXX-XXXX`, signs in (or signs up)
  in the browser if needed, confirms the code matches the terminal, and
  approves. The CLI then stores the session and the command completes on its
  own. Codes expire after 15 minutes; a denied or expired approval fails the
  command with a clear error. This flow also works on OIDC/SSO-only
  deployments, because authentication happens in the browser.

- **Interactive email/password.** In a TTY, plain `kora auth login` prompts
  for credentials.

When driving the CLI as a coding agent: run `kora auth login` (pass
`--base-url` or set `KORA_BASE_URL` — there is no prompt in a non-TTY shell),
surface the printed URL and confirmation code to the user, and wait for the
command to finish. Confirm with `kora auth whoami`; the session is stored on
disk and every later `kora` command picks it up automatically.

Notes:

- A brand-new user does not need `kora auth signup` first: the device
  verification page redirects to the web login, which links to sign-up and
  returns to the approval page afterwards. `kora auth signup` (TTY-only,
  local email/password accounts) remains available for interactive use.
- On a local-mode self-managed install, the first signed-up user becomes
  platform admin. Server installs require SSO sign-in with one of the configured
  verified platform-admin bootstrap emails; unverified local-auth first-user
  bootstrap is limited to local single-machine evaluation.
- Fully headless CI with no human in the loop: inject a session via
  `KORA_SESSION_JSON_B64` — see `references/auth-and-sessions.md`.

## First Session, End To End

```sh
npm install -g @kora-platform/cli
kora auth login --base-url http://localhost:3000   # prints a browser URL; approve the login there
kora org create --name "My Org" --slug my-org
kora status
```

`kora org create` sets the new organization active. With access to multiple
organizations, switch with `kora org select <org>` and check with
`kora org current` or `kora auth whoami`.

## Working Conventions

- Append `--json` for machine-readable output. Discover the surface with
  `kora help --json` and `kora help <command-path> --json` instead of
  guessing flags.
- Most commands require an active organization.
- Destructive commands (`kora org delete`, `kora org reset`, and similar)
  prompt for confirmation unless `--yes` is passed.
- Organization API keys (`kora access api-keys create`) authenticate direct
  Platform API HTTP calls; they are not a CLI login method in this release.

## Which Reference To Read Next

- `references/auth-and-sessions.md` — session storage and permissions,
  non-interactive session injection, base-URL precedence detail, OIDC
  behavior, and API keys
- `references/command-map.md` — top-level command families and aliases
