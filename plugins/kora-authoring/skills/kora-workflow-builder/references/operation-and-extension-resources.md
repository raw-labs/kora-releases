# Operations And Extension Calls

Operations are named service-script callables used by process `service` nodes.
They run TypeScript, JavaScript, Python, shell, or another command in the
runtime sandbox and emit one JSON output value through `@kora/runtime-sdk`.

This reference owns operation YAML and script-to-extension wiring. Before
choosing an installed extension function, read
`references/installed-extension-discovery.md` and fetch the exact selected
function contract with `kora extensions get`.

## Operation

- file path: `operations/<name>.yaml`
- kind: `Operation`
- purpose: define one script-backed service call

```yaml
apiVersion: kora/v1
kind: Operation
metadata:
  name: lookup-company
spec:
  description: Look up company details
  script:
    command: node
    args: ["scripts/lookup-company.ts"]
    parseStdoutAsJson: true
  bindings:
    extensions:
      crm:
        name: crm-primary
        functions: ["lookupCompany"]
  paramBindings:
    domain:
      from: input
      path: $.domain
  resultMapping:
    company:
      from: $.stdout.company
  timeoutMs: 30000
  retry:
    maxAttempts: 3
    initialIntervalMs: 1000
    backoffMultiplier: 2
    nonRetryableErrors: ["invalid_input"]
```

Process nodes reference operations by name:

```yaml
- id: lookup
  type: service
  operation: lookup-company
  input: CompanyLookupInput
  output: CompanyLookupOutput
  next: done
```

## Operation-To-Extension Calls

To call an installed extension from a service operation:

1. Discover and inspect the exact function contract first.
2. Declare `spec.bindings.extensions.<alias>.name`.
3. List only the selected extension functions under `functions`.
4. In the script, call
   `extensions.invoke(alias, functionName, input)` and `emitOutput(value)`.
5. Use `resultMapping` to map emitted JSON into the service node's typed
   output.
6. Smoke test the service node with
   `kora test node <workflow-name> <node-id> --workspace <dir> --input @<dir>/input.json --environment <environment> --json`,
   or run from the workspace directory and use
   `--workspace . --input @input.json --environment <environment>`.

Do not author placeholder extension calls. If the required extension is missing,
disabled, not setup, or lacks the selected function in the target environment,
stop and report that setup issue before creating or publishing a runnable
workflow. Do not replace a missing connected extension with direct provider
`fetch()` calls unless the user explicitly changes the requirement to use a
public unauthenticated API.

The alias is local to the operation. The script sees only the selected function
names and typed input/output; extension secrets and storage stay inside
extension code. `name` references an installed extension in the target
environment, not extension package source in the workflow project.

Missing installs, missing functions, and missing grants appear as
release-readiness or deployment-readiness diagnostics during release creation,
`kora release validate <release> --environment <environment> --json`,
workflow-node tests, and environment deploy.

For script input/output examples, read `references/service-io.md`. For
runtime variables and org-secret bindings, read `references/credentials.md`.
