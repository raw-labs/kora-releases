# Manifest And Entrypoint

## Contents

- Manifest shape and permissions
- Entrypoint module shape
- Host registrations and hooks
- Lifecycle hooks and domain hook events
- Ephemeral output rules

## Manifest

```yaml
apiVersion: kora/v1
kind: ExtensionPackage
metadata:
  name: github
  description: GitHub integration
spec:
  version: 1.0.0
  entrypoint: src/index.ts
  permissions:
    storage: true
    secrets: true
    callbacks: true
    schedules: true
    cloud: true
  limits:
    timeoutMs: 300000
```

Supported permissions are `storage`, `secrets`, `callbacks`, `schedules`, and
`cloud`. `cloud` exposes `ctx.cloudBaseUrl` and `ctx.cloudToken` only when the
package declares it and the install grants it; offline/manual-license
deployments may still receive `null` values.

The manifest declares requested permission limits. Grants are install-owned
approval; do not hard-code approval into package source.

## Entrypoint

Extension source imports schemas from `@kora/extension-sdk` and exports a
default registration function:

```ts
import { Type } from "@kora/extension-sdk";

export default function extension(kora) {
  // registrations go here
}
```

The extension SDK is `@kora/extension-sdk`. It exports `Type` and `Static` from
TypeBox.

```ts
import { Type, type Static } from "@kora/extension-sdk";

const LeadInput = Type.Object(
  { company: Type.String({ minLength: 1 }) },
  { additionalProperties: false }
);
type LeadInput = Static<typeof LeadInput>;

export default function register(kora) {
  kora.registerFunction({
    name: "summarizeLead",
    description: "Summarize a lead record for follow-up.",
    input: LeadInput,
    output: Type.Object({ summary: Type.String() }, { additionalProperties: false }),
    async run(input) {
      const lead = input as LeadInput;
      return { summary: `${lead.company} should be followed up.` };
    }
  });
}
```

## Host Surface

The `kora` host supports:

- lifecycle hooks: `onInstall`, `onEnable`, `onDisable`, `onGrant`,
  `onDelete`, `onSetupReady`
- domain hooks: `extensionPackages.onPublished`, `releases.onValidate`,
  `workflows.onRunStarted`, `workflows.onRunCompleted`,
  `humanTasks.onNotification`, `humanTasks.onCreated`,
  `humanTasks.onCompleted`, `humanTasks.onTimeout`
- registrations: `registerFunction`, `registerFunctionProvider`,
  `registerTool`, `registerToolProvider`, `registerSkill`,
  `registerSettingsView`, `registerCallback`, `registerSchedule`

Hooks receive `(event, ctx)`. `event.name` is one of `install`, `enable`,
`disable`, `delete`, `grant`, `setupReady`, `packagePublished`,
`releaseValidate`, `humanTaskNotification`, `humanTaskCreated`,
`humanTaskCompleted`, `humanTaskTimeout`, `workflowRunStarted`, or
`workflowRunCompleted`; `event.payload` is optional.

### Lifecycle Hooks

Lifecycle hooks are install-local host callbacks for Platform install state
changes. Setup expectations are listed per hook.

| Method | Event | Platform timing | Setup contract |
| --- | --- | --- | --- |
| `kora.onInstall(handler)` | `install` | After install commits. Payload includes `packageArtifactId`. | Setup may still be required; initialize only install-local state that tolerates missing provider configuration. |
| `kora.onEnable(handler)` | `enable` | After a disabled install is enabled. | Setup may still be required; resume only local state that does not require provider credentials. |
| `kora.onDisable(handler)` | `disable` | After an install is disabled. | Setup may still be required; pause local state without assuming provider credentials. |
| `kora.onGrant(handler)` | `grant` | After permission grants are saved. Payload includes `grantedPermissions`. | Setup may still be required; grants do not imply provider credentials are configured. |
| `kora.onDelete(handler)` | `delete` | After delete commits and before install-scoped safe storage and runtime state are cleared. Runtime state includes callback registrations, schedule registrations, and registration snapshots. | Setup may never have completed. Best-effort cleanup may inspect own state but must tolerate missing setup; failures are recorded and do not abort delete. |
| `kora.onSetupReady(handler)` | `setupReady` | After `ctx.setup.markReady()` is accepted and committed. Payload includes `packageArtifactId`. | Setup is complete for that transition. This hook may run again if setup is later marked required and completed again. |

Use domain hooks for Platform events such as package publication, release
validation, workflow runs, and human task notifications.

## Ephemeral Outputs

`registerFunction`, `registerFunctionProvider`, `registerTool`, and
`registerToolProvider` default to stored outputs. Use
`outputPersistence: "ephemeral"` only for short-lived credentials or bearer
values that the current runtime call may consume but Kora must not persist or
replay. Ephemeral outputs still require an output schema when the caller
depends on typed output.

Never log, throw, audit, or return bearer values outside the ephemeral result
object.
