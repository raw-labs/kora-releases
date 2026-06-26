# Credentials

Runtime scripts do not read extension storage or extension secrets directly.
They call extension functions exposed through the operation runtime SDK binding.
For organization-owned secrets, operations declare a named org-secret binding
and the runtime injects the resolved value into the declared environment
variable for that script invocation only.

Rules:

- Extension handlers own provider-specific storage, refresh, OAuth callback,
  API key, and token logic.
- Kora Cloud-backed built-ins are still extensions. Discover their registered
  functions and tools with `kora extensions search` and exact
  `kora extensions get`, then call them with `extensions.invoke(alias,
  functionName, input)`.
- Operation scripts receive only the values a registered function explicitly
  returns.
- Prefer extension functions that perform the provider action directly. A
  credential-returning helper is only appropriate when the discovered function
  contract intentionally returns short-lived credential material to the current
  runtime call.
- Operation `spec.bindings.secrets` declares environment-scoped org secrets for
  the service script environment. Use scalar form when the Settings secret name
  is also the injected environment variable name, or `{ name, as }` when they
  differ. The YAML stores only the secret name, never the value.
- Operation `spec.bindings.env` declares non-secret runtime variables for the
  service script environment using the same scalar or `{ name, as }` shape.
- Runtime-variable names, org-secret names, and injected env names must not use
  the reserved `KORA_` prefix.
- Extension safe-storage secrets are separate from org secrets. Extension code
  can use `ctx.secrets` for its own install-scoped secrets, but extensions do
  not receive org-secret values.
- Short-lived credential material is redacted from logs, telemetry, and error
  text. Operation stdout and mapped results are not automatically redacted; do
  not emit credential material there unless it is the intentional operation
  result.
- If the script needs a non-secret runtime value, pass it through operation
  input or runtime variables instead of hardcoding it.
