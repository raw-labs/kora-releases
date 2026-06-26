# Lifecycle, Diagnostics, And Configuration

## Start

```sh
./koractl start
```

`start` pulls first-party images unless `KORA_SKIP_PULL=1`, renders the
OpenSandbox config, validates license files before mutating Compose state,
runs migrations, starts API, workers, web, and Caddy, then waits for
`/healthz` to become reachable before returning. Run it after any
configuration or image change; it is idempotent.

## Stop And Restart

```sh
./koractl stop
./koractl restart [service]
```

`stop` also runs OpenSandbox child-container cleanup for sandbox containers
with host mounts under the configured workspace directory. `restart` without
arguments restarts the whole stack.

## Inspect

```sh
./koractl status
./koractl logs platform-api
./koractl doctor
```

`status` summarizes the stack as `stopped`, `starting`, `healthy`, or
`unhealthy`, then prints the underlying Compose service status and the basic
HTTP health result. `logs` without a service name tails the whole stack.
`doctor` checks Docker, Compose, env placeholders, license files,
deployment-token permissions, basic disk/RAM capacity, public URL
consistency, and whether Compose state can be inspected — and prints
remediation text instead of raw stack traces. Reach for `doctor` first when
anything looks wrong.

## Model Configuration

```sh
./koractl configure models
```

Updates only `KORA_CHAT_MODEL`, `KORA_CHAT_MODEL_API_KEY`,
`KORA_AGENT_MODEL`, and `KORA_AGENT_MODEL_API_KEY`. The chat model powers
user-facing conversations in the Kora web UI. The agent model powers Kora agent
execution work, such as coding tasks and automation. Operators can use the same
model and API key for both, or split them for different cost, latency, or
capability settings. `KORA_AGENT_MODEL` is the default for capabilities that
omit `agentConfig.model`; explicit per-capability model refs are enabled per
environment in Platform Settings > Agent models. Existing model API keys are
preserved by default and never printed as prompt defaults; leave a key prompt
blank only to keep the current value. New installs must provide model API keys
before `./koractl install` or `./koractl start` can continue. Get keys from the
selected provider's account/API-key page and save default keys only through the
scoped `KORA_CHAT_MODEL_API_KEY` and `KORA_AGENT_MODEL_API_KEY` settings.

## Access Configuration

```sh
./koractl configure access
```

Updates the login allowlist and, for OIDC-enabled installs, platform-admin
bootstrap emails. For demo/server deployments, replace
`KORA_AUTH_ALLOWED_EMAILS=*` with exact emails and/or domains:

```sh
KORA_AUTH_ALLOWED_EMAILS=alice@example.com,@example.com
```

## Advanced Files

Operators may inspect or manually edit `compose.yaml`, `.env`, and
`.rendered/opensandbox.toml`. `./koractl` renders `.rendered/opensandbox.toml`
before invoking Compose; do not copy it over the bundle template
`opensandbox.toml`. `./koractl configure` preserves unknown `.env` keys where
practical. `./koractl compose <args...>` passes through to Docker Compose for
advanced support; prefer the named commands in documentation and day-to-day
operation.
