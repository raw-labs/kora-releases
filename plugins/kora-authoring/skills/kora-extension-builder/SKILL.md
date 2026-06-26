---
name: kora-extension-builder
description: >
  Read when creating, changing, validating, or publishing Kora extension package
  source, or when explaining the handoff from a published extension package
  to Settings installation and configuration.
---

# Kora Extension Builder

Use this skill when the user wants to create or change extension package source.
Extension packages are separate source bundles with their own lifecycle; they
are not workflow release-source files.

When the extension targets a named third-party product, vendor, API, or public
service, research first. In Kora chat, use `web_search` and then `web_fetch` on
the most relevant official docs or product pages before choosing endpoints,
auth style, request fields, or response fields. Do not design provider-specific
behavior from model memory alone. If web research is unavailable, fails, or does
not find useful official sources, say that plainly and either ask for a docs URL
or continue only with clearly labeled assumptions and a generic adapter shape.

An authoring agent can create and edit extension package source in the
workspace, validate it with `kora extensions validate <path> --json`, and
publish it with `kora extensions publish <path> --json` when the user asks to
publish.
Install, permission grants, removal, built-in installation, secrets,
OAuth/callback setup, and extension settings UI are deployment operations, not
extension package source authoring.

## Mental Model

An extension package workspace contains:

```text
extension.yaml
src/**
skills/**
assets/**
README.md
```

`extensions publish` stores an immutable published package. Extension install,
grant, enable/disable, configuration, and removal are environment-scoped
Settings actions.

The manifest declares requested permission limits. Grants are install-owned
approval; do not hard-code approval into package source.

## Build Flow

When building an extension package, follow this order:

1. Decide what the extension is for: static callable functions, static agent
   tools, dynamic agent tools, install settings, public callbacks, schedules,
   bundled runtime skills, hooks, or a combination of those surfaces. For named
   third-party systems, do the official-source web research above before
   drafting package source.
2. Create a package directory in the workspace, normally under
   `extensions/<package-name>/`.
3. Write `extension.yaml` with package metadata, metadata-only version,
   entrypoint, requested permissions, and timeout limits.
4. Write `src/index.ts` with a default registration function that imports
   `Type` from `@kora/extension-sdk` and registers the needed surfaces.
5. Add package-local skill files under `skills/<skill-name>/SKILL.md` only when
   the installed extension should provide bundled runtime skills.
6. Add package assets under `assets/**` when handlers or bundled skills need
   immutable package-local files.
7. Run `kora extensions validate extensions/<package-name> --json`.
8. Fix validation diagnostics until validation returns `ok: true`.
9. When the user wants an installable package, run
   `kora extensions publish extensions/<package-name> --json`.
10. Tell the user the package is published but not installed yet, then provide
    the Settings route from `kora-product-ui` for installation and setup.

Do not skip from invalid source to install. `extensions publish` runs package
validation again and only stores an immutable published package if the package
is valid.

## Authoring Commands

Use these package lifecycle commands while building package source:

- `kora extensions validate <package-dir> --json`
- `kora extensions publish <package-dir> --json`

Use installed-extension discovery only when package work needs to compare
against an already installed extension:

- `kora extensions search --environment <environment> --name "<wildcard>" --json`
- `kora extensions search --environment <environment> --description "<wildcard>" --json`
- `kora extensions get <extension-name> --environment <environment> --json`

Use progressive disclosure for installed extensions:

1. Search installed extensions by name, title, or description.
2. If the needed extension is absent, search available extensions with
   `kora extensions search --environment <environment> --scope available --name "<wildcard>" --json`
   or `--description "<wildcard>"`; use the returned `nextCommands.install`
   only when the user explicitly wants an install/deployment operation.
3. Once one installed extension is selected, search inside it:
   `kora extensions search <extension-name> --environment <environment> --kind function --description "<wildcard>" --json`.
   The `--kind` value can be `function`, `tool`, `skill`, `function-provider`,
   or `tool-provider`. Use `--name` for exact or wildcard names and
   `--description` for intent/topic search.
4. Fetch the exact contract only for the selected function/tool/skill:
   `kora extensions get <extension-name> --environment <environment> --function <name> --json`.

For a ready installed extension, use `kora extensions invoke <extension-name>
<function-name> --environment <environment> --input @input.json --yes --json`
only when the package task explicitly needs to compare or inspect current
installed behavior. Do not use it as package validation, install setup, or a
replacement for `kora extensions validate`.

After publish in the product chat surface, read `kora-product-ui` and route
installation/setup to its Extensions settings destination. `kora-product-ui`
owns exact Platform UI routes.

Do not mix package authoring with install/configuration changes. In product
chat, route install, grant, enable, disable, delete, and configure actions to
Settings. In an external terminal-agent context, use the full `kora` CLI only
when the user explicitly asks for those deployment operations and the CLI is
authenticated to the target deployment.

## Minimal Function Package

Use this shape when the extension adds a stable callable capability to an
installed environment, such as normalizing a record, generating a summary, or
wrapping a safe API call.

```text
extensions/lead-helper/
  extension.yaml
  src/index.ts
```

```yaml
apiVersion: kora/v1
kind: ExtensionPackage
metadata:
  name: lead-helper
  description: Lead utility functions for workflow service nodes.
spec:
  version: 0.1.0
  entrypoint: src/index.ts
```

```ts
import { Type, type ExtensionHost, type Static } from "@kora/extension-sdk";

const LeadInput = Type.Object(
  { company: Type.String({ minLength: 1 }) },
  { additionalProperties: false }
);
type LeadInput = Static<typeof LeadInput>;

export default function extension(kora: ExtensionHost) {
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

Build and publish it with:

```sh
kora extensions validate extensions/lead-helper --json
kora extensions publish extensions/lead-helper --json
```

## Common Extension Shapes

- Static function package: registers `registerFunction` for workflow service
  nodes and agents to call after install.
- Agent tool package: registers `registerTool` when agents should see a
  concrete tool with a fixed schema.
- Dynamic function provider package: registers `registerFunctionProvider` when
  workflow-callable functions depend on install settings, secrets, storage, or
  an external catalog discovered at runtime.
- Dynamic tool provider package: registers `registerToolProvider` when agent
  tools depend on install settings, secrets, storage, or an external catalog
  discovered at runtime.
- Settings-backed connector: registers `registerSettingsView` plus functions to
  test credentials, save setup state, or open an external setup URL. Use
  `secrets` permission for credentials and keep setup in Settings. Call
  `kora.configureSetup({ required: true })` when setup must complete before
  workflow runtime use. Required setup stays blocked until a setup-safe settings
  function or callback completes setup and calls `ctx.setup.markReady()`.
  Disconnect/reset paths that invalidate shared external credentials must call
  `ctx.setup.markRequired()`; use `ctx.setup.markRequired({ scope: "artifact" })`
  only when invalidation is specific to the current package artifact.
- Callback connector: registers `registerCallback` when an external service
  must call Kora back. Use `callbacks` permission and callback helpers for URLs,
  state, metadata, and HMAC verification. Mark only callbacks that are part of
  required setup as `setup: true`; normal webhook/action callbacks should not be
  setup-safe.
- Scheduled extension: registers `registerSchedule` for recurring extension
  work. Use `schedules` permission and do not start unmanaged background loops.
- Skill package: registers `registerSkill` and includes
  `skills/<root>/SKILL.md` when the installed extension should provide bundled
  runtime skills.

## Validation Summary

`kora extensions validate <path> --json` is the package preflight. It checks:

- package file safety and size limits, manifest shape, and entrypoint existence;
- SDK-typed TypeScript source and registration loading through the trusted
  runtime;
- registration schema shape and cross-references, registered skill files, static
  `outputPersistence` values, registered handler presence, and Ajv-compiled JSON
  Schemas;
- settings-view render output, using in-memory storage, missing secret metadata,
  fake callback metadata, null Cloud context, and blocked network fetch. Render
  handlers must show an unconfigured setup state without contacting external APIs.

Dynamic function and tool providers are validated for registration shape at
package validation time. Their runtime `list` output is validated when Platform
needs to populate a missing provider-owned capability surface in the install
registration snapshot. Discovery is cache-first; empty provider `list` results
are rejected and not persisted.

`kora extensions publish <path> --json` runs the same package validation
before storing an immutable published package. Publish is the strong
server-side gate, not just a file upload. Use it only after validation succeeds
and the user wants a package they can install.

Package validation proves the extension package loads and registers valid
metadata and that registered settings views return valid declarative JSON. It
does not prove every external API path, credential, OAuth callback, or dynamic
provider tool succeeds at runtime. Test external behavior through Settings,
callbacks, capability discovery after setup, and extension-backed workflow-node
tests after install.

`kora test node <workflow-name> <node-id> --workspace <dir> --input @input.json --json`
validates and executes workflow service-node code after an extension is
installed and a service node is bound to an operation that uses it. It does not
validate raw extension package source by itself.

## Authoring Rules

- Keep package source outside workflow release source.
- Validate before publishing.
- Publish before the user installs through Settings.
- Ask the user to review and grant only permissions within the package-declared
  limits in Settings.
- For settings views, define shared button constants and pass them both in
  static `registerSettingsView({ submit, buttons })` metadata and in the
  declarative JSON returned from `render`.
- Use settings view `description`, short `blocks`, field `description`, and
  text/textarea `placeholder` values for setup guidance. Keep labels short and
  mark required fields with `required: true` instead of writing "required" into
  the label.
- Do not call external APIs from settings view render handlers. Use render for
  local setup/status display, and use settings functions/buttons for external
  test, save, OAuth, or provider actions.
- Treat Platform setup lifecycle separately from provider-specific status text.
  Settings views may render "pending", "connected", or provider errors, but
  durable workflow readiness is changed only with `ctx.setup.markReady()` and
  `ctx.setup.markRequired()` from successful handlers. `ctx.setup.markRequired()`
  defaults to install-wide invalidation for shared external credentials.
- Browser callback completion only tells the Settings panel to refresh; it does
  not make setup ready unless the callback handler itself calls
  `ctx.setup.markReady()` after verifying the provider state.
- Treat `ctx.actor` as optional. Callback, schedule, lifecycle, and runtime
  hook handlers must be able to run without a user actor, and storage/secrets
  writes are attributed by the host rather than by extension-supplied user ids.
- Use `outputPersistence: "ephemeral"` for bearer-value outputs only.
- Use operation runtime SDK bindings when workflow service scripts need to call
  installed extension functions.

## Which Reference To Read Next

- `references/manifest-and-entrypoint.md` — manifest, entrypoint, lifecycle
  hooks, domain hooks, registrations, and ephemeral output rules
- `references/functions-and-tools.md` — static functions, static tools, dynamic
  tool providers, provider result schemas, refresh behavior, and collision rules
- `references/settings-callbacks-schedules.md` — bundled runtime skills,
  settings views, form fields, buttons, callbacks, OAuth-style state, HMAC,
  schedules, and human-task notifications
- `references/extension-context.md` — handler context, actor shape, storage,
  secrets, callbacks, locks, schedules, events, audit, same-install function
  invocation, and extension internal isolation
