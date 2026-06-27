---
name: kora-self-host
description: >
  Read when installing, configuring, operating, updating, or debugging a
  self-managed Kora Platform deployment: the install.sh bootstrap, koractl
  lifecycle commands (install, configure, start, stop, status, logs, doctor,
  update), install modes, and license activation.
---

# Kora Self-Host Operations

Use this skill when the user wants to install Kora on a machine they control,
or to operate an existing self-managed deployment. Using a running Kora from
the terminal (signup, login, organizations, workflows) is the `kora-cli`
skill, not this one.

## Mental Model

A self-managed Kora deployment is a Docker Compose bundle driven entirely by
the single `./koractl` command from the install directory. Operators do not
run `docker compose` directly except through the advanced escape hatch.

Supported platforms: macOS with Docker Desktop or a compatible Docker/Compose
setup, and Linux with Docker Engine plus the Docker Compose plugin. V1 does
not support native Windows, Dockerless runtimes, or bundled native
Postgres/Redis/Temporal/OpenSandbox. The installer fails with remediation
text when Docker or the Compose plugin is missing.

Install directory layout (bundle files plus runtime-generated files):

```text
kora-platform/
  koractl
  compose.yaml
  .env.example
  Caddyfile
  opensandbox.toml
  init-db.sql
  LICENSE.md
  COPYRIGHT
  THIRD_PARTY_NOTICES.md
  .env             # generated configuration
  license/         # license material and deployment token
  state/           # chat workspaces, workflow artifacts
  .rendered/       # rendered OpenSandbox config
```

Do not call Docker cleanup commands directly; `koractl` owns lifecycle details
such as OpenSandbox cleanup and is the supported operator surface.

`license/license-deployment-token` is a bearer secret: keep it owner-readable
only and never log it, commit it, or send it to support. The signed
`license.json` is entitlement material, not a bearer secret.

In the kora-platform source repository, `pnpm dev:up` is the repo-development
path. Never use it for self-managed installs, and never point users at this
skill for repo development.

## Install

The public bootstrap downloads the latest release bundle, verifies its
checksum, extracts it to a temporary directory, and delegates target selection
and installation to the bundled `./koractl bootstrap-install`:

```sh
curl -fsSL https://kora.raw-labs.com/install.sh | bash
```

Online installs (Free, Pro, Teams, and account-managed Enterprise) use an
install session owned by Kora Accounts. Without an install-session token, the
terminal prints a Kora Accounts URL; the human completes sign-in, plan
selection, and checkout in the browser while the terminal polls. When Kora
Accounts approves the session, the installer writes the license material and
starts Kora. When driving this as a coding agent: run the command, surface
the printed URL to the user, and wait — browser cancellation or expiry is
reported back to the terminal instead of hanging until the polling timeout.

Pick an install mode up front:

- **local** — one person, one machine. Defaults to
  `$HOME/.kora/kora-platform` and `http://localhost:3000`, plain HTTP, first
  signed-up user becomes platform admin.
- **server** — a VM or host reachable by a team. Defaults to
  `/opt/kora-platform`, HTTPS via Caddy, and requires a login allowlist.
  Server installs require verified SSO admin bootstrap; unverified local-auth
  first-user bootstrap is limited to local single-machine evaluation.
- **offline** — air-gapped Enterprise/evaluation; license files arrive out of
  band and the deployment never contacts Kora Accounts.

For non-interactive automation, pass flags to the bootstrap or set the
matching environment variables:

```sh
curl -fsSL https://kora.raw-labs.com/install.sh | bash -s -- --mode server --dir /opt/kora-platform
```

`--mode` answers the local/server/offline prompt; `--dir` answers the
install-directory prompt. `KORA_INSTALL_MODE` and `KORA_INSTALL_DIR` are the
environment equivalents. `KORA_NONINTERACTIVE=1` makes prompts use defaults
or provided environment values, but a useful fully non-interactive install
still needs explicit values for required server, model, access, and license
configuration.

For a server install without prompts, provide the mode, directory, public
hostname, model settings, verified SSO admin bootstrap config, login allowlist,
and either an install-session token or a plan to complete the printed browser
approval. Server mode requires OIDC or `local+oidc`; local email/password
first-user bootstrap is only for local single-machine evaluation.

```sh
export KORA_NONINTERACTIVE=1
export KORA_PUBLIC_DOMAIN=kora.example.com
export KORA_AUTH_MODE=oidc
export KORA_BOOTSTRAP_ADMIN_EMAILS=admin@example.com
export KORA_AUTH_ALLOWED_EMAILS=admin@example.com,@example.com
export KORA_OIDC_ISSUER=https://idp.example.com
export KORA_OIDC_CLIENT_ID=kora-platform
export KORA_OIDC_CLIENT_SECRET="$OIDC_CLIENT_SECRET"
export KORA_OIDC_REDIRECT_URI=https://kora.example.com/api/v1/auth/oidc/callback
export KORA_OIDC_SCOPES=openid,email,profile
export KORA_OIDC_PROVISIONING=allowlist_user
export KORA_CHAT_MODEL_INPUT=anthropic/claude-opus-4-6
export KORA_CHAT_MODEL_API_KEY_INPUT="$MODEL_PROVIDER_API_KEY"
export KORA_AGENT_MODEL_INPUT=anthropic/claude-opus-4-6
export KORA_AGENT_MODEL_API_KEY_INPUT="$MODEL_PROVIDER_API_KEY"

curl -fsSL https://kora.raw-labs.com/install.sh | \
  bash -s -- --mode server --dir /opt/kora-platform --install-session kora_ins_...
```

`KORA_CHAT_MODEL_INPUT` and `KORA_AGENT_MODEL_INPUT` must be supported
`koractl` Pi model choices. Use `anthropic/claude-opus-4-6` unless the
operator explicitly chooses another listed model. The chat model powers
user-facing conversations in the Kora web UI; the agent model powers Kora agent
execution work, such as coding tasks and automation. Operators can use the same
model and API key for both, or split them for different cost, latency, or
capability settings. `KORA_AGENT_MODEL` is the default for capabilities that omit
`agentConfig.model`; explicit per-capability model refs are enabled per
environment in Platform Settings > Agent models. The matching default model API
keys are required before install or start can continue; get them from the
selected provider's account/API-key page, such as Anthropic, OpenAI, Google,
xAI, Groq, OpenRouter, Mistral, DeepSeek, Moonshot, or Z.ai.

Reruns into an existing install directory must state intent with
`--existing-action` or `KORA_EXISTING_INSTALL_ACTION` (one of `start`,
`update`, `configure`, `relink`, `reinstall`, `cancel`). Other bootstrap
flags are `--install-session <token>`, `--cloud-base-url <url>`, and
`--version <release>`.

Mode defaults, install sessions, manual bundle installs, and the required
configuration checklist are in `references/install.md`.

## Verify

After install or any lifecycle change:

```sh
./koractl status
./koractl doctor
```

`status` summarizes the stack as `stopped`, `starting`, `healthy`, or
`unhealthy`, then prints Compose service status and the basic HTTP health
result. `doctor` checks Docker, Compose, env placeholders, license files,
deployment-token permissions, disk/RAM capacity, and public URL consistency,
and prints remediation text instead of stack traces. A healthy local install
serves the web app at `http://localhost:3000`.

## Command Table

```sh
./koractl install
./koractl configure
./koractl configure models
./koractl configure access
./koractl configure license
./koractl start
./koractl stop
./koractl restart [service]
./koractl status
./koractl logs [service]
./koractl doctor
./koractl update [version]
./koractl license status
./koractl license activate
./koractl license install-file
./koractl version
```

Advanced support can use `./koractl compose <docker-compose-args...>`, but
prefer the named commands.

## Handoff To Product Use

Once `./koractl status` reports healthy, account and workspace work moves to the
web app at the deployment base URL or to the Kora CLI — read the `kora-cli`
skill. On a local-mode install, the first user to sign up becomes platform
admin. On server installs, sign in with SSO using one of the configured verified
platform-admin bootstrap emails.

## Which Reference To Read Next

- `references/install.md` — mode defaults, online install sessions, manual
  release installs, existing-install reruns, and the required configuration
  checklist
- `references/operate.md` — start/stop/restart semantics, status, logs,
  doctor, model and access configuration, and advanced files
- `references/update-and-license.md` — updates, manual rollback, license
  operations, and deployment-token hygiene
