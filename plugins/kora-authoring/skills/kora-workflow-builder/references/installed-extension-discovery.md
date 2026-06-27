# Installed Extension Discovery

Use this reference before authoring workflow operations or agent capabilities
that depend on an already-installed extension. Extension package source is
owned by `kora-extension-builder`; this file is only about discovering what a
target environment already exposes.

Use one resolved environment for every installed-extension command in a task.
If the user named an environment, use it. Otherwise resolve the environment
from `kora environment list --json` before searching. Do not mix extension
searches, contract fetches, node tests, release validation, deployment, or run
commands across different environments.

## Progressive Discovery

1. Find candidate installed extensions:
   `kora extensions search --environment <environment> --name "<wildcard>" --json`
   or `kora extensions search --environment <environment> --description "<wildcard>" --json`.
2. If the extension is missing and the user needs setup options, search
   available built-ins:
   `kora extensions search --environment <environment> --scope available --name "<wildcard>" --json`
   or `--description "<wildcard>"`. Route installation and configuration to
   Settings unless the user explicitly asked for CLI deployment work.
   Stop after reporting the missing install/setup. Do not keep building a
   runnable workflow with placeholder calls, guessed function names, or direct
   provider API fallbacks.
3. Inspect the selected installed extension before binding it:
   `kora extensions get <extension-name> --environment <environment> --json`.
   Continue only when `state` is `ready`. If it is
   `setup_required`, stop and tell the user to complete setup in Settings. If
   it is `disabled`, stop and tell the user the extension must be enabled. Use
   the returned `message` as the user-facing explanation.
4. After selecting one ready installed extension, search inside the selected surface:
   `kora extensions search <extension-name> --environment <environment> --kind function --description "<wildcard>" --json`.
   Use `--kind tool` for agent tools and `--kind skill` for agent guidance:
   `kora extensions search <extension-name> --environment <environment> --kind tool --description "<wildcard>" --json`
   or
   `kora extensions search <extension-name> --environment <environment> --kind skill --description "<wildcard>" --json`.
   Use `--name` for exact or wildcard names and `--description` for
   intent/topic search.
5. Fetch exactly one selected function/tool/skill contract:
   `kora extensions get <extension-name> --environment <environment> --function <name> --json`,
   `--tool <name>`, or `--skill <name>`.

Use the returned function description and input schema as the source of truth.
Treat the output schema as an expected shape, not a guarantee: optional
extension fields may be absent or `null`, so write operation stdout and
`resultMapping` defensively around the exact fields the workflow needs. Do not
infer behavior from function names alone.

## One-Off Function Probes

Use `kora extensions invoke` only after progressive discovery selected one
ready installed extension and `kora extensions get ... --function <name>
--json` returned the exact schema.

Write probe input to a temporary JSON file and invoke the exact function:

```sh
kora extensions invoke <extension-name> <function-name> --environment <environment> --input @input.json --yes --json
```

Use this only when the user needs current data from a connected provider before
authoring a workflow. It is not workflow verification and it does not test
service-node bindings, `paramBindings`, `resultMapping`, or released source.
Use `kora test node` for authored workflow-node behavior.

Before invoking any function that may send messages, publish content, create
records, update data, delete data, or change a connected service, tell the user
what the function may do and ask for explicit confirmation. Do not invoke setup
functions or settings actions from workflow authoring; route setup to Settings.

Do not publish a workflow expected to fail because an extension is not ready
unless the user explicitly asks to publish anyway with that warning.
Provider-specific setup/status views can explain what is missing, but Platform
lifecycle state is the authoring gate.

If a required provider extension is absent, disabled, or `setup_required`, the
correct next step is user/operator setup, not a workaround. Do not use public
or unauthenticated provider APIs to bypass a missing extension when the user
asked for organization/provider data that requires the connected extension.

## Example: List GitHub Pull Requests

If the user asks to list pull requests in GitHub, search progressively:

```sh
kora extensions search --environment production --name "*github*" --json
kora extensions get github --environment production --json
kora extensions search github --environment production --kind function --description "*pull request*" --json
kora extensions search github --environment production --kind function --name "*PULL*" --json
kora extensions get github --environment production --function GITHUB_LIST_PULL_REQUESTS --json
```

Then bind only the selected exact function name in the operation YAML and call
that function from the service script.

If no installed GitHub extension is found, search setup options:

```sh
kora extensions search --environment production --scope available --name "*github*" --json
```

Route installation and provider connection setup to Settings unless the user
explicitly asked for CLI deployment work.
