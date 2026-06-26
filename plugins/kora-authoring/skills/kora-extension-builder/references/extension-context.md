# Extension Context

## Contents

- Handler context and actor shape
- Storage and secrets
- Callback, lock, schedule, setup lifecycle, event, and audit helpers
- Same-install function invocation
- Extension internal isolation

## Handler Context

Handlers receive `ctx` with:

- `ctx.orgId`, `ctx.installId`, `ctx.environmentId`, `ctx.environmentKey`,
  `ctx.executionId`, `ctx.idempotencyKey`, `ctx.packageHash`, `ctx.cloudBaseUrl`,
  `ctx.cloudToken`, and optional `ctx.actor`
- `ctx.actor.userId` and `ctx.actor.roles` when a user actor is available
- `ctx.storage.get/set/delete/list` for per-install durable JSON state
- `ctx.secrets.write/read/metadata/delete` for per-install secrets backed by
  safe storage
- `ctx.callbacks.createUrl/createState/consumeState/metadata/verifyHmacSignature`
  for callback URL, state, metadata, and HMAC helpers
- `ctx.locks.withInstallLock(name, run)` for install-scoped critical sections;
  this requires the `storage` permission
- `ctx.schedules.metadata(name)` for schedule metadata
- `ctx.setup.markReady()` and `ctx.setup.markRequired()` for Platform-owned
  install readiness transitions after setup actions or disconnect/reset actions
- `ctx.events.publish(name, payload)` for extension-authored events
- `ctx.audit.record(eventName, payload)` for redacted audit records
- `ctx.invokeFunction(name, input)` for same-install function calls

`ctx.actor` is optional. It is present for user/request-originated invocations
when Platform has a request actor to expose, and absent for system-originated
callbacks, schedules, lifecycle hooks, runtime-event hooks, and validation
renders. Do not require `ctx.actor` in handlers that can be triggered by an
external callback or Platform background work.

Use `ctx.idempotencyKey` as the stable duplicate-detection key for
side-effecting per-invocation external calls when it is present. Do not use
`ctx.executionId` as a provider idempotency key; it identifies the extension
execution record and may change if an upstream caller retries with a new bridge
request.

## Storage And Secrets

Extension storage and secrets are per install. Use storage for durable JSON
state such as cursors, external ids, or connection status. Use secrets for API
keys, OAuth tokens, private keys, and other credential material.
Storage and secret writes are attributed by the Extension Host using actor
id/type metadata. Extension code does not pass a user id to `ctx.storage` or
`ctx.secrets`, and system-originated writes are recorded as system actors rather
than fake users.

Do not put raw credential values in manifests, settings status rows, audit
payloads, thrown errors, logs, or stored function/tool outputs. Use
`outputPersistence: "ephemeral"` only for bearer-value outputs that must be
returned to the immediate caller.

## Callbacks And Locks

Use `ctx.callbacks.createUrl`, `createState`, and `consumeState` for callback
URLs and OAuth-style state. Use `ctx.callbacks.metadata` when a handler needs
callback endpoint metadata, and `ctx.callbacks.verifyHmacSignature` for signed
callbacks.

Use `ctx.locks.withInstallLock(name, run)` around install-scoped critical
sections such as token refresh, provider catalog refresh, or connection
promotion. Do not implement lock files or unmanaged background loops.

## Schedules, Events, And Audit

Use `ctx.schedules.metadata(name)` to inspect schedule registration metadata.
Use `ctx.events.publish(name, payload)` for extension-authored events. Use
`ctx.audit.record(eventName, payload)` for redacted audit records when a
settings action, callback, or privileged function changes install state.

## Setup Lifecycle

Use `ctx.setup.markReady()` only after the current handler has completed the
setup required for normal workflow runtime use. Use `ctx.setup.markRequired()`
from disconnect, reset, or credential-clearing handlers that make the install
unsafe for normal runtime use. `markRequired()` defaults to `{ scope: "install" }`,
clearing setup completion for all artifacts of that install because the shared
backing credential is invalid. Use
`ctx.setup.markRequired({ scope: "artifact" })` only when the setup invalidation
is specific to the current package artifact. These calls update Platform
lifecycle after the handler succeeds; if the handler throws, the setup transition
is not persisted.

For lifecycle hook timing and setup contracts, see
`manifest-and-entrypoint.md#lifecycle-hooks`.

Keep provider-specific status in settings view output. Status labels such as
"pending", "connected", or "invalid credentials" help users understand the
provider, but workflow readiness is controlled by Platform lifecycle.

Browser callback completion pages and settings-view refreshes are not lifecycle
transitions. A callback that completes setup must be registered with
`setup: true`, verify the callback state/provider condition, and call
`ctx.setup.markReady()` itself. Callback, disconnect, and reset handlers should
use install-scoped `ctx.setup.markRequired()` when they clear or invalidate the
configured provider state.

## Same-Install Invocation

Use `ctx.invokeFunction(name, input)` when one extension handler needs to call a
function registered by the same install. This is useful for settings buttons,
callbacks, schedules, hooks, and notification handlers that should reuse a
registered function path.

## Isolation Boundary

Extension secrets and storage stay inside extension code. User-authored
operation scripts can call exposed extension functions through
`@kora/runtime-sdk` bindings, but cannot read extension internals.

Operations declare explicit installed-extension function bindings. The runtime
SDK grants the script only the allowed function invocation path, not extension
storage, extension secrets, settings views, callbacks, schedules, or package
source.
