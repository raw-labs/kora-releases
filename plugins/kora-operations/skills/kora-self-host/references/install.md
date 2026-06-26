# Install Modes, Sessions, And Required Configuration

## Mode Defaults

### Local Computer

For one person installing Kora on their own machine:

1. install directory: `$HOME/.kora/kora-platform` when using the public
   bootstrap
2. base URL: `http://localhost:3000`
3. chat sandbox API URL: `http://host.docker.internal:3000` on Docker
   Desktop, or the Docker bridge gateway such as `http://172.17.0.1:3000` on
   native Linux
4. Caddy site address: `http://localhost`, plus
   `KORA_CADDY_ADDITIONAL_SITE_ADDRESS` set to the sandbox-visible host for
   local sandbox requests, so local browser and sandbox requests stay plain HTTP
5. auth cookies are marked non-secure because local mode is plain HTTP
6. the first signed-up local user becomes platform admin
7. login allowlist defaults to `*`
8. workflow artifacts are stored under `<install-dir>/state/artifacts`

### Server / Team

For a VM or host reachable by a team:

1. install directory: `/opt/kora-platform`
2. `KORA_PUBLIC_BASE_URL=https://<domain>`
3. `KORA_CHAT_API_BASE_URL=https://<domain>` unless a sandbox-visible override is needed
4. `KORA_CADDY_SITE_ADDRESS=<domain>`, so Caddy terminates TLS on ports 80
   and 443
5. verified SSO admin bootstrap is required; unverified local-auth first-user
   bootstrap is rejected
6. OIDC-enabled installs require OIDC provider config and verified
   platform-admin bootstrap emails
7. an explicit auth allowlist (server installs do not accept the local `*`
   default)

The installer does not configure DNS. Point `KORA_PUBLIC_DOMAIN` at the VM
before the final smoke test. Non-interactive server installs must set
`KORA_PUBLIC_DOMAIN`; the installer will not continue with the
`platform.example.com` placeholder.

### Offline Enterprise / Evaluation

Offline mode never contacts Kora Accounts, but it still needs a network
profile for the local web app: `KORA_OFFLINE_NETWORK_MODE=server` with
`KORA_PUBLIC_DOMAIN=<hostname>` for a VM, or `KORA_OFFLINE_NETWORK_MODE=local`
for a local evaluation. The operator receives an offline enterprise package
out of band containing the release bundle, registry or preloaded-image
instructions, a signed `license.json`, and trusted
`license-public-keys.json`. Install the license files, then start:

```sh
./koractl license install-file --license ./license.json --keys ./license-public-keys.json
./koractl start
```

## Online Install Sessions

Kora Accounts owns account, plan, billing, deployment identity, and
install-session approval; the local installer owns machine setup. Starting
`./koractl install` without an install-session token prints the Kora Accounts
URL; the human completes sign-in, plan selection, and checkout in the
browser, and the terminal polls until Kora Accounts approves the session,
then writes the license material and starts Kora.

Website-first installs carry an install-session token so the browser flow is
not repeated:

```sh
curl -fsSL https://kora.raw-labs.com/install.sh | bash -s -- --install-session kora_ins_...
```

Resume an already-installed bundle with a token:

```sh
./koractl license activate --install-session kora_ins_...
```

If `KORA_CLOUD_BASE_URL` was recorded during install, `koractl` reuses that
same Kora Accounts host for later `license activate` retries.

or via environment variables:

```sh
KORA_CLOUD_BASE_URL=https://kora.raw-labs.com KORA_INSTALL_SESSION_TOKEN=kora_ins_... ./koractl license activate
```

Use the matching Kora Accounts host for non-production environments, for
example `https://kora-staging.raw-labs.com/install.sh`.

## Manual Release Install

Deploy bundles are published to the public `raw-labs/kora-releases` artifact
repository:

```sh
tag=0.2.0-rc1
curl -LO "https://github.com/raw-labs/kora-releases/releases/download/${tag}/kora-platform-deploy-${tag}.tar.gz"
curl -LO "https://github.com/raw-labs/kora-releases/releases/download/${tag}/kora-platform-deploy-${tag}.tar.gz.sha256"
sha256sum -c "kora-platform-deploy-${tag}.tar.gz.sha256"
mkdir -p /opt/kora-platform
tar -xzf "kora-platform-deploy-${tag}.tar.gz" --strip-components=1 -C /opt/kora-platform
cd /opt/kora-platform
./koractl install
```

To pin or roll back through the bootstrap instead, pass `--version` after
`bash -s --`.

## Existing Installs

The public bootstrap delegates existing-install handling to
`./koractl bootstrap-install` after the target directory is selected. It detects
an existing install when that directory already has `.env` or license material.
Interactive runs ask what the operator intended: start, update, reconfigure,
relink the license/account, reinstall after backups, or cancel.
Non-interactive reruns must set `KORA_EXISTING_INSTALL_ACTION` or pass
`--existing-action` with one of `start`, `update`, `configure`, `relink`,
`reinstall`, `cancel`.

Normal reruns should use the named lifecycle commands directly instead of
re-running install:

```sh
./koractl start
./koractl update
./koractl configure models
./koractl license activate --install-session kora_ins_...
```

## Non-Interactive Inputs

`--mode <local|server|offline>` and `--dir <path>` are bootstrap flags; pass
them after `bash -s --` so the installer does not ask the install-mode or
install-directory questions. The matching environment variables are
`KORA_INSTALL_MODE` and `KORA_INSTALL_DIR`.

`KORA_NONINTERACTIVE=1` suppresses prompts by using defaults or provided
environment values, but it is not a complete configuration by itself. Server and
offline-server installs still require a real `KORA_PUBLIC_DOMAIN`. Server-style
access validation requires `KORA_AUTH_MODE=oidc` or `local+oidc`, a non-`*`
`KORA_AUTH_ALLOWED_EMAILS`, `KORA_BOOTSTRAP_ADMIN_EMAILS`, and the minimum
OIDC provider config: `KORA_OIDC_ISSUER`, `KORA_OIDC_CLIENT_ID`,
`KORA_OIDC_CLIENT_SECRET`, and `KORA_OIDC_REDIRECT_URI`. Provide those through
the prompt inputs `KORA_ALLOWED_EMAILS_INPUT` and `KORA_ADMIN_EMAILS_INPUT` plus
environment variables during install, or preseed `.env` before running
`./koractl install`.
If `KORA_OIDC_PROVISIONING` is omitted, the installer persists `allowlist_user`
so the verified bootstrap admin email can create the first Platform user.
Explicit `invite_only` is rejected for installer-driven admin bootstrap.

Model prompts can be answered with:

```sh
KORA_CHAT_MODEL_INPUT=anthropic/claude-opus-4-6
KORA_CHAT_MODEL_API_KEY_INPUT=...
KORA_AGENT_MODEL_INPUT=anthropic/claude-opus-4-6
KORA_AGENT_MODEL_API_KEY_INPUT=...
```

`KORA_CHAT_MODEL_INPUT` and `KORA_AGENT_MODEL_INPUT` must be supported
`koractl` Pi model choices. Use `anthropic/claude-opus-4-6` as the default
unless the operator explicitly chooses another listed model.

The chat model powers user-facing conversations in the Kora web UI. The agent
model powers Kora agent execution work, such as coding tasks and automation.
Operators can use the same model and API key for both, or split them for
different cost, latency, or capability settings.

`KORA_AGENT_MODEL` is the default for capabilities that omit
`agentConfig.model`. Explicit per-capability model refs are enabled later per
environment in Platform Settings > Agent models by saving that model's API key.

Model API keys are required before install or start can continue. Create them
from the account/API-key page for the selected provider, such as Anthropic,
OpenAI, Google, xAI, Groq, OpenRouter, Mistral, DeepSeek, Moonshot, or Z.ai.
`KORA_SKIP_MODEL_CONFIG=1` is only valid when the environment or `.env` already
has complete model values, including both API keys.
Set `KORA_SKIP_ACCESS_CONFIG=1` only when access values are already seeded.

Bootstrap flags:

- `--mode <local|server|offline>` — selects install mode and default network
  behavior.
- `--dir <path>` — selects the install directory.
- `--install-session <token>` — skips creating a new browser approval session
  and claims the provided Kora Accounts install session.
- `--cloud-base-url <url>` — points install-session approval at a non-default
  Kora Accounts host, for example staging.
- `--version <release>` — installs that release instead of latest.
- `--existing-action <start|update|configure|relink|reinstall|cancel>` —
  states what to do when the target directory already has install state.

## Required Configuration

`./koractl install` creates `.env` from `.env.example`, generates deployment
secrets, asks for local/server/offline mode, and can collect model and access
settings. It stops before license claim or startup when model API keys are
missing. At minimum, a working online deployment needs:

1. signed license files
2. `KORA_CHAT_MODEL`
3. `KORA_CHAT_MODEL_API_KEY`
4. `KORA_AGENT_MODEL`
5. `KORA_AGENT_MODEL_API_KEY`
6. `KORA_AUTH_ALLOWED_EMAILS`
7. platform-admin bootstrap settings

OIDC/SSO, external S3-compatible artifact storage, customer-owned OTLP
telemetry, and data contribution are intentionally not first-run installer
prompts. Online Free/Pro/Teams installs configure account-service telemetry
automatically; Enterprise/evaluation/offline installs keep account-service
telemetry and data contribution inactive by default.
