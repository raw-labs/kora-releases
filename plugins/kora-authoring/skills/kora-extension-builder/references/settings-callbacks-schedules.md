# Settings, Callbacks, Schedules, And Bundled Skills

## Contents

- Bundled runtime skills
- Settings views, form fields, and buttons
- Public callbacks, OAuth-style state, and HMAC verification
- Schedules
- Human-task notification payloads

## Bundled Runtime Skills

Extension packages may register bundled runtime skills for an installed
extension:

```ts
kora.registerSkill({
  name: "github-triage",
  description: "Use when triaging GitHub issues.",
  path: "skills/github-triage"
});
```

The `path` is a package-relative skill root, and the root must contain
`SKILL.md`. Adjacent files such as `references/**`, templates, examples, or
images are part of the immutable package artifact.

This is distinct from the public Kora agent plugin/package mechanism. Extension
runtime skills are bundled inside an extension package and become available
through an installed extension artifact; they are not a separate public plugin
artifact.

## Settings Views

`registerSettingsView` takes `{ name, title, render, input?, submit?,
buttons?, permission?, timeoutMs? }`. `render(input, ctx)` returns one
declarative view:

- form: `{ type: "form", title, description?, blocks?, fields, submit, buttons? }`
- status: `{ type: "status", title, description?, blocks?, rows: [{ label, value }], buttons? }`
- table: `{ type: "table", columns, rows, description?, blocks?, buttons? }`

`description` is a short view-level summary. `blocks` are small declarative
content blocks for setup instructions or sectioning:

- section: `{ type: "section", title, description? }`
- text: `{ type: "text", text, tone?: "default" | "muted" | "warning" }`

Form fields are:

- text or textarea: `{ name, type: "text" | "textarea", label, description?, placeholder?, required?, value? }`
- secret: `{ name, type: "secret", label, description?, required? }`
- select: `{ name, type: "select", label, description?, options: [{ label, value }], required?, value? }`
- checkbox: `{ name, type: "checkbox", label, description?, value? }`

Use field `description` for help text instead of putting instructions into the
label. Use `required: true` for required fields; the UI supplies the required
visual convention. Keep blocks short and static enough to be useful in Settings
without becoming provider-specific custom UI.

Settings buttons are `{ function, label, style?, actionHint? }`, where `style`
is `primary`, `secondary`, or `danger`. Buttons reference registered extension
functions by name and may include
`actionHint: { type: "openUrl", target?: "newTab" | "sameTab" }`.

Form submit buttons receive all current form field values. Secondary form
buttons receive only the top-level form fields declared by their target
function's input schema when that schema is a closed object
(`additionalProperties: false`); closed empty-object schemas receive `{}`. If
the target function uses an open or non-object schema, Kora preserves the full
form payload for compatibility.

Buttons are validated twice: static settings view metadata authorizes which
functions the Settings page may submit, and the render handler returns the
declarative buttons to display. Use shared constants so those two places cannot
drift:

```ts
const saveConnection = { function: "saveConnection", label: "Save", style: "primary" as const };
const testConnection = { function: "testConnection", label: "Test", style: "secondary" as const };

kora.registerSettingsView({
  name: "connection",
  title: "Connection",
  submit: saveConnection,
  buttons: [testConnection],
  async render(_input, ctx) {
    const current = await ctx.storage.get<{ serverUrl?: string }>("connection");
    return {
      type: "form",
      title: "Connection",
      description: "Save the endpoint and credential used by this extension install.",
      blocks: [
        { type: "section", title: "Endpoint" },
        { type: "text", text: "Test the connection before marking setup ready.", tone: "muted" }
      ],
      fields: [
        {
          name: "serverUrl",
          type: "text",
          label: "Server URL",
          description: "Use the base URL for the service, without a trailing path.",
          placeholder: "https://api.example.com",
          required: true,
          value: current?.serverUrl ?? ""
        },
        {
          name: "apiKey",
          type: "secret",
          label: "API key",
          description: "Stored as a secret input and redacted from logs."
        }
      ],
      submit: saveConnection,
      buttons: [testConnection]
    };
  }
});
```

Settings view render handlers must be able to render the unconfigured setup
state during package validation. Do not call external APIs from render. Put
external checks, saves, OAuth starts, and provider handoffs behind settings
functions referenced by `submit` or `buttons`.

Use `registerSettingsView` plus functions for settings-backed connectors that
need to test credentials, save setup state, or open an external setup URL. Use
the `secrets` permission for credentials and keep setup in Settings.
Read-only status and table views may include `refresh: { intervalMs }` for
bounded polling. Editable form views do not auto-refresh. Browser callback
completion only refreshes the initiating Settings panel; it does not imply setup
success unless the callback handler verifies the provider state and calls
`ctx.setup.markReady()`.

## Callbacks

`registerCallback` takes `{ name, run, input?, permission?, setup?, timeoutMs? }`.
`run(request, ctx)` receives `{ method, headers, query, body?, rawBodyBase64? }`
and returns `{ type: "done", message? }`, `{ type: "redirect", url }`, or
`{ type: "json", status?, body }`.

Use `ctx.callbacks.createState`, `ctx.callbacks.createUrl`, and
`ctx.callbacks.consumeState` for OAuth-style state handling. Use
`ctx.callbacks.verifyHmacSignature` for signed callbacks.

Use `registerCallback` when an external service must call Kora back. Declare
the `callbacks` permission and use callback helpers for callback URLs, state,
metadata, and HMAC verification.

Set `setup: true` only for callbacks that complete required setup, such as an
OAuth return callback used by the setup view. Setup callbacks may run while the
install is `setup_required`; normal webhook/action callbacks are blocked until
setup is ready. A setup callback that invalidates or clears configuration should
call `ctx.setup.markRequired()`, which invalidates setup for the whole install by
default. Use `{ scope: "artifact" }` only for current-artifact-specific setup
invalidations.

## Schedules

`registerSchedule` takes `{ name, cron, run, permission?, timeoutMs? }`.
`run(event, ctx)` receives `{ scheduledAt }`. Schedules are Temporal-backed;
do not start unmanaged background loops from extension code.

Use `registerSchedule` for recurring extension work. Declare the `schedules`
permission.

## Human-Task Notifications

`kora.humanTasks.onNotification(handler)` receives payload fields including
`orgId`, `workflowId`, `taskId`, `reviewUrl`, `expiresAt`, environment
deployment ids, optional `assigneeId`, optional `assigneeEmail`, optional
`dueAt`, and optional `taskSummary`.

Notification handlers must treat environment context as part of the delivery
target so staging and production human-task notifications cannot be confused.
