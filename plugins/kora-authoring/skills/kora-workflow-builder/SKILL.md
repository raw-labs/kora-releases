---
name: kora-workflow-builder
description: >
  Read when creating or changing Kora workflow release source: org-model
  resources, capabilities, operations, decisions, processes, service scripts,
  runtime SDK calls, managed artifacts, validation, release creation, or
  deployment follow-up. Not for extension package authoring, product UI route
  lookup, platform inspection, or exact schema lookup.
---

# Kora Workflow Builder

Use this skill when the user wants to create or change workflow release source:
add a role, add a person, introduce a capability, restructure resources, design
a process, create an operation, connect service code to an installed extension,
or make a workflow handle runtime inputs, outputs, and managed artifacts.

Describe planned and completed work in product terms -- what the workflow does --
not filenames, paths, YAML kinds, script languages, or runtime SDK wiring, unless
the user asks for implementation detail or you are explaining a failure.

Keep validation and repair loops internal: report the product behavior, the
checks that passed, and only the diagnostics that need user input.

Do not offer provider-specific destinations such as Slack or email unless the
user named that provider, extension discovery shows the resolved environment
supports it, or the user is explicitly planning missing extension setup. Before
discovery, use generic destinations: run output, a modeled human task, or an
installed extension if available.

This skill is a table of contents. Read the smallest reference that answers the
next question. For exact resource fields, use `kora schema get <resource>
--json`; the CLI schema wins over prose. Schema resource names are lowercase
registry names such as `process`, `operation`, `capability`, and `decision`,
not YAML `kind` values.

Fresh workflow source has a schema preflight. Before writing a new bundle from
an empty workspace, either start from the complete minimal service workflow in
`references/patterns-and-examples.md` or query the current schemas first:
`manifest`, `organization`, `assignments`, `operation`, and `process`. Query
`role`, `person`, `agent`, `capability`, and `decision` as needed before using
those resources. Use `kora schema get <schema-name> --json` once per schema.
The schema name for `kora.yaml` is `manifest`, not `project`.

## Core Mental Model

The chat source workspace is the source editing area for workflow source files. It may start empty.
The first durable server-side artifact is created with
`kora release create <dir> --json`. Release creation does not create a live
deployment. The target product path is release creation followed by explicit
environment deployment.

Default source resolution:

1. If the workspace already has source files, inspect and edit that workspace
   source.
2. For fresh creation with no existing workspace source, author in the chat
   workspace.
3. For read, add, or modify requests against existing product behavior or
   source, do not guess from an empty workspace. Resolve the intended
   environment from the user request first. If the user names `test`, `staging`,
   `production`, or another environment, use it. Otherwise inspect
   `kora environment list --json`. If exactly one environment exists, use it as
   the default resolved environment; if multiple environments exist, ask which
   environment before materializing release source.
4. For the resolved environment, inspect
   `kora deployment list --environment <environment> --json`.
5. If it has a live deployment, use that live deployment's release as the default
   source and write the frozen source into the workspace with
   `kora release source <release> --out <empty-temp-dir> --json`. Then copy the
   exported files into the chat workspace root before editing.
6. Do not keep edits under temporary folders such as `source/`, `src/`, or
   `project/`; edit workspace-root paths like `org/...`, `processes/...`, and
   `scripts/...` directly.
7. If it has no live deployment, say there is no live release to modify and
   continue only with fresh source or a user-named release.

For read-only inspection of already-created workflow or org-model facts, use
the CLI's release snapshot selectors instead of materializing source:
`--release <release>` for a named immutable release, or
`--environment <environment>` for the selected environment's current live
deployment release.

Every workflow source bundle has six layers you may touch:

1. who does the work: people, agents, roles, assignments
2. what work exists: capabilities and operations
3. how work flows: processes
4. what external behavior exists: installed extensions, operations, and service scripts
5. what rules apply: decisions
6. what runtime assets support execution: templates, scripts, schemas, docs

Human and agent task nodes depend on this runtime chain:

`role -> capability -> assignment -> process task`

If any link is inconsistent, the workflow will not resolve at runtime.
Service nodes are different: they call an `Operation` and do not need roles,
capabilities, agents, or assignments unless the workflow also has human/agent
task nodes. Still create `org/assignments.yaml` for every fresh source bundle; use an
empty `spec.roles: {}` registry when the workflow has no human or agent tasks.

## Runtime Boundary

- The workspace is the proposal surface for authored source -- YAML resources,
  service scripts, templates, and source docs. It is not the live runtime; use
  it for source editing and smoke tests only.
- Extension package source uses the `kora-extension-builder` skill and the
  extension lifecycle, not workflow release source.
- Temporary files, generated outputs, and smoke-test artifacts may exist while
  authoring, but only files accepted by release creation become deployment
  entries.
- Files provided during a conversation are unclassified workspace/input files;
  use them as message context. Copy one into release source only when the user
  explicitly asks; upload it as a managed runtime artifact only when the user
  explicitly asks to use it as workflow/run input.
- `kora` is an authoring and inspection tool, not available inside live
  workflows, operations, scripts, or agent prompts. Do not put `kora` commands
  in workflow YAML, scripts, operations, or agent prompts.
- Service operation scripts run in a separate runtime sandbox with released
  files mounted read-only at `/workspace`. Package installs, node_modules,
  virtualenvs, caches, and build output created while authoring are scratch
  state unless the source bundle has an explicit deterministic packaging
  strategy.

## Environment Discipline

Resolve the intended environment once (see Core Mental Model). Use that same
environment for extension discovery, contract fetches, node tests, release
validation, deployment, and run commands. Do not discover an extension in one
environment and test, release, deploy, or run against another.

For extension-backed service work:

- Verify the selected extension is installed in the resolved environment and
  that `kora extensions get ... --json` reports `state: "ready"`.
- If it is `setup_required`, route the user to Settings setup before publishing
  or running workflow code. If it is `disabled`, tell the user it must be
  enabled. Use the returned `message` as the user-facing explanation.
- If a required provider-backed extension is not installed and ready, stop and
  tell the user exactly which extension/setup is missing. Do not write
  placeholder provider operations, do not substitute unauthenticated direct API
  calls, and do not publish a workflow you expect to fail unless the user
  explicitly asks to continue after that warning.
- To inspect current provider data before authoring, invoke one ready installed
  extension function (`kora extensions invoke <extension-name> <function-name>
  --environment <environment> --input @input.json --yes --json`) after fetching
  its exact schema. Do not create throwaway workflow source just to inspect
  provider data.

## Service Operation Rules

Script-backed service nodes invoke an `Operation`.

- Service scripts import `@kora/runtime-sdk` and use `getInput`,
  `extensions.invoke`, and `emitOutput` for operation input, installed
  extension functions, and stdout JSON. Cloud-backed providers are still
  installed extensions from the script perspective.
- For installed extension calls, normalize the extension response into the
  operation's own stdout shape; do not rely on optional extension fields being
  present or non-null.
- Registered extension function descriptions and schemas are the source of truth
  when authoring operations that call them.
- `paramBindings` shape operation stdin before the script runs.
- Runtime variables and org secrets are declared with `spec.bindings.env` and
  `spec.bindings.secrets`; names and injected env vars must not use the
  reserved `KORA_` prefix.
- Operation scripts can call functions exposed through
  `spec.bindings.extensions`, but they do not read extension storage or
  extension secrets directly.
- When `spec.script.parseStdoutAsJson` is true or omitted, stdout must contain
  exactly one valid JSON value. Log diagnostics to stderr.
- `resultMapping` maps emitted stdout into the service node's declared output.
  Without `resultMapping`, script stdout must be a JSON object that can shallow
  merge into workflow state.
- Optional `resultMapping` fields are skipped only when the source path is
  absent. A present `null` value is mapped and must validate.
- Managed artifact files are separate from stdout. Scripts read artifact inputs
  and write declared artifact outputs through `KORA_ARTIFACT_MANIFEST`
  `localPath` entries; do not persist manifest local paths into workflow state.

## Typical Project Layout

```text
kora.yaml
org/
  org.yaml
  roles/<name>.yaml
  people/<name>.yaml
  agents/<name>.yaml
  assignments.yaml
capabilities/
processes/
operations/
decisions/
scripts/
templates/
```

Keep entity names in resource `metadata`, not inferred from filenames.

## Build Order

When creating source from scratch or deliberately restructuring it, prefer
this order so references resolve cleanly. Complete the fresh workflow source
schema preflight before writing the first file.

1. source manifest (`kora.yaml`) and organization (`org/org.yaml`)
2. `org/assignments.yaml` with `spec.roles: {}` for every new source bundle
3. roles, people, agents, and assignment entries only when human/agent tasks need them
4. capabilities
5. installed extensions, operations, and service scripts
6. decisions
7. processes
8. optional templates and supporting docs

## Authoring Discipline

Before editing:

- Inspect the current workspace before assuming files exist.
- Check `kora.yaml` and the target area directly before scaffolding foundational
  files.
- `read` or `ls` the area you plan to change.
- `grep` for references before renaming or deleting anything.

While editing:

- When the user gives a clear intent but omits incidental details, draft a valid
  minimal workflow with reasonable names and a simple message/manual trigger
  instead of stopping for clarification.
- Keep changes small and scoped.
- For processes, author the data contract first. Define `types:` and use only
  the node fields the schema supports.
- Treat workflow state as an accumulated top-level object. Each typed node
  output shallow-merges into state.
- Use message starts for external inbound triggers. A workflow invoked by
  `call` or `call.each` needs a callable manual start marked `internal: true`.
- Use `task.each` or `call.each` for runtime collections.
- For agent tasks that process file content, use node-level `promptAttachments`
  on `task` / `task.each` for selected top-level file artifact input fields.
  Prefer an earlier extractor service node for PDFs, Office docs, HTML, ZIPs, or
  other binary/rich attachments; attach only the extracted text or image
  artifact to the agent prompt.
- Make operation `resultMapping` and the declared process service-node
  `output` type agree.
- Model expected absence explicitly. Optional means absent from the object, not
  `null`, unless the schema deliberately allows `null`.

After meaningful changes:

- Run `kora test node <workflow-name> <node-id> --workspace <dir> --input @input.json --environment <environment> --json`
  for touched executable service nodes when you can supply meaningful node
  input.
  Use this before claiming artifact-producing nodes are verified; release
  creation and deployment do not prove artifact capture.
- Use `kora release create <dir> --json` only when the user asks to create a
  release artifact from source. Then:
  1. Treat release-source or release-readiness diagnostics from that command as
     must-fix items before deployment.
  2. When it succeeds, tell the user the release was created, that it is not live
     yet, and route them with a Markdown link such as
     `[release detail](/app/releases/<release>)`.
  3. Inspect `kora environment list --json` before offering deployment follow-up.
     If exactly one environment is available, default to that named environment
     and ask whether to deploy there. If multiple environments are available, ask
     which environment should receive the release.
  4. Use `kora release validate <release> --environment <environment> --json` to
     check one environment's readiness.
  5. Use `kora environment deploy <environment> <release> --json` only when the
     user asks to deploy a release, and name the target environment in your
     confirmation.
- If a workflow-node smoke test fails, do not publish or deploy unless the user
  explicitly says to proceed anyway after you warn that the workflow is likely
  to fail. Classify the failure before responding: source/validation error,
  missing extension/binding/grant, missing connection/credential, extension
  runtime error, or provider/API error.
- For operations that call installed extension functions, use the visible
  installed-extension discovery commands. Use the `kora-extension-builder`
  skill only when a new extension package source is needed. Release creation
  does not publish, install, or grant extensions; extension package and
  install lifecycle actions are admin/product workflows outside workflow source.
- Do not ask the user to test anything you can safely test yourself in the
  current authoring environment.
- If testing requires user authorization, missing business input, or real
  changes in connected systems, say exactly what remains untested and what user
  action is needed.
- Do not claim release readiness until the requested release creation or
  explicit release validation is clean. Release creation is not a substitute
  for workflow-node smoke tests.
- When reporting success to the user, keep checks and user-visible behavior in
  the foreground. Avoid implementation bullets for scripts, YAML files, runtime
  SDK calls, or command mechanics unless the user asks for them.

Cascading/destructive changes:

- Identify dependents first with `grep`.
- Tell the user what will be affected.
- Get explicit confirmation before destructive removals.
- Update references in one pass, then validate.

Reminder: real org membership, runtime facts, workflow runs, tasks, releases,
runtime variables, workflow artifacts, and resource schemas are not in the
workspace. Use `kora` for those.

## Which Reference To Read Next

- `references/org-model-resources.md` — modeled people, roles, agents, and
  assignments
- `references/capability-resource.md` — capability resource shape
- `references/process-flow-nodes.md` — process node types and flow rules
- `references/patterns-and-examples.md` — common workflow patterns and one
  complete current project example
- `references/installed-extension-discovery.md` — find installed extensions,
  search functions/tools/skills, and fetch one exact callable contract
- `references/operation-and-extension-resources.md` — operations, service
  scripts, and runtime SDK bindings for selected extension functions
- `references/service-io.md` — operation stdin, stdout, `emitOutput`, and
  `resultMapping`
- `references/artifacts.md` — managed artifact declarations, manifests, and
  send-template artifact URLs
- `references/credentials.md` — runtime variables, org-secret bindings, and
  extension secret boundaries
- `references/filesystem.md` — live runtime filesystem and dependency
  availability
- `references/testing.md` — workflow-node and artifact smoke-test routing
- `references/decision-resource.md` — decision resource shape
- `references/agent-config.md` — agent-specific modeling
- `references/yaml-resource-schemas.md` — YAML shape companion

## Typical Task Routing

- "Where should this resource live?" -> `references/org-model-resources.md`
- "How should this workflow be structured?" ->
  `references/process-flow-nodes.md`, then
  `references/patterns-and-examples.md`
- "How does a workflow call external behavior?" ->
  `references/installed-extension-discovery.md`, then
  `references/operation-and-extension-resources.md`, then `references/service-io.md`
- "What extension function/tool/skill should this workflow use?" ->
  `references/installed-extension-discovery.md`
- "How should this service script read input or emit output?" ->
  `references/service-io.md`
- "How should this workflow use files or folders at runtime?" ->
  `references/artifacts.md`, then `references/filesystem.md`
- "How should this node be tested?" -> `references/testing.md`
- "What exact fields does resource X support?" ->
  `kora schema get <resource> --json`
- "What is already modeled in this source proposal?" -> inspect the workspace
  files directly

## Out Of Scope

- Extension package authoring, publishing, installing, updating, and granting ->
  the `kora-extension-builder` skill.
- Product UI route lookup, menus, tabs, buttons, and user navigation ->
  the `kora-product-ui` skill.
- Platform inspection -> `kora` with the matching family, driven by the
  bootstrap source-of-truth map.
- Exact resource fields -> `kora schema get <resource> --json`, not skill
  prose.
