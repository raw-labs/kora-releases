# Sessions, Non-Interactive Use, And API Keys

## Device Login Approval

`kora auth login --device` (automatic in non-TTY shells) asks the deployment
for a device login session, prints the verification URL
(`<base-url>/device?code=XXXX-XXXX`) plus the confirmation code, and polls
until the login is approved in a browser:

1. The user opens the URL. If they are not signed in to the web app, they are
   redirected to the login page (with a sign-up link) and returned to the
   approval page afterwards.
2. The approval page shows the confirmation code and the account that will be
   used. Approving binds the CLI session to that account; denying fails the
   waiting command.
3. The CLI receives the same session material as a password login and stores
   it in the session file. Nothing else changes downstream.

Properties worth knowing:

- Confirmation codes are short-lived (15 minutes), single-use, and stored
  hashed server-side; the polling secret never appears in the browser URL.
- Approvals and denials are recorded as audit events, and the resulting login
  is audited like any other.
- The flow works on OIDC/SSO-only deployments, since the browser handles
  authentication.

## Session Storage

A successful `kora auth login` or `kora auth signup` writes the session to
`$XDG_STATE_HOME/kora/session.json`, defaulting to
`~/.local/state/kora/session.json`. The file holds the access token, refresh
token, user identity, active organization, and the base URL — treat it as a
secret. It is written with mode `0600`, and the CLI refuses to read a
session file with broader permissions.

`kora auth whoami` uses the stored session and active organization, then
fetches the current user from the deployment API. `kora auth logout` clears
the session and revokes the refresh token when possible.

## Non-Interactive Session Injection

For CI or headless environments where no interactive login happened on the
same machine, inject a session through the `KORA_SESSION_JSON_B64`
environment variable: the session JSON, base64url-encoded.

1. Log in once on a trusted machine.
2. Encode the session file:

   ```sh
   node -e 'console.log(Buffer.from(require("fs").readFileSync(process.argv[1], "utf8"), "utf8").toString("base64url"))' ~/.local/state/kora/session.json
   ```

3. Provide the value as `KORA_SESSION_JSON_B64` to the environment running
   `kora`.

The injected session carries real tokens; scope it like any credential
(masked CI secret, never committed).

## Base URL Configuration Files

`.kora/cli.toml` (found by walking up from the working directory, so it can
be checked into a project to pin that project's deployment) and the global
`~/.config/kora/config.toml` share the same two keys:

```toml
baseUrl = "http://localhost:3000"
openBrowser = false
```

Environment (`KORA_BASE_URL`) overrides the repo file, which overrides the
global file. Login persists an interactively chosen base URL into the global
config so later logins do not re-prompt.

## OIDC / SSO Deployments

When a deployment enables OIDC alongside local auth, interactive login and
signup print the SSO URL and continue with email/password. When a deployment
is OIDC-only, interactive `kora auth login` and `kora auth signup` stop and
point at `kora auth login --device` instead: device approval happens in the
browser, where SSO works, so it is the supported CLI auth path on OIDC-only
deployments.

## Organization API Keys

API keys are org-scoped credentials for calling the Platform HTTP API
directly — the supported non-browser auth path for programmatic API access:

```sh
kora access api-keys create ci-runner
kora access api-keys list
kora access api-keys revoke <key-id>
```

The key secret is returned once at creation. API keys do not log the CLI in;
CLI commands authenticate with the stored session.
