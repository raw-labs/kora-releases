# YAML Resource Schemas

This is a compact companion. The CLI schema is authoritative:

```bash
kora schema get <resource> --json
```

Schema resource names are lowercase registry names. They are not YAML `kind`
values. For example, use `kora schema get manifest --json` for `kora.yaml`;
`project` is not a schema name.

## Resource Locations

| Resource | Schema name | Kind | Path |
| --- | --- | --- | --- |
| Project manifest | `manifest` | `Project` | `kora.yaml` |
| Organization | `organization` | `Organization` | `org/org.yaml` |
| Role | `role` | `Role` | `org/roles/*.yaml` |
| Person | `person` | `Person` | `org/people/*.yaml` |
| Agent | `agent` | `Agent` | `org/agents/*.yaml` |
| Assignments | `assignments` | `Assignments` | `org/assignments.yaml` |
| Capability | `capability` | `Capability` | `capabilities/*.yaml` |
| Process | `process` | `Process` | `processes/*.yaml` |
| Operation | `operation` | `Operation` | `operations/*.yaml` |
| Decision | `decision` | `Decision` | `decisions/*.yaml` |

Extension package manifest YAML belongs to the `kora-extension-builder` skill,
not this org project schema reference.

Most org resource documents use this outer shape:

```yaml
apiVersion: kora/v1
kind: ResourceKind
metadata:
  name: resource-name
spec: {}
```

`Process` is different: it has top-level `start`, `flow`, and optional `types`
fields instead of a `spec` wrapper.

`Project` is also different: it lives in `kora.yaml` and uses the project
manifest schema.

```yaml
apiVersion: kora/v1
kind: Project
metadata:
  name: acme-review
spec:
  machineExecution:
    defaults:
      sandbox:
        network:
          defaultAction: deny
```

Use `spec.machineExecution.defaults` for shared execution defaults such as
sandbox policy. Agent task extension tools and skills are enabled on the
capability with `spec.agentConfig.extensions`, either by shorthand extension name
or by object-form `install` plus `tools`/`skills` selections.

## Operation

Operations are script-backed service calls.

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
```

## Process Service Node

```yaml
- id: lookup
  type: service
  operation: lookup-company
  input: CompanyLookupInput
  output: CompanyLookupOutput
  next: done
```

`input` and `output` refer to process `types:` entries. Operation
`resultMapping` must produce fields that satisfy the service node's `output`
type.

`task` and `task.each` nodes may declare whole-node retry:

```yaml
retry:
  maxAttempts: 2
  initialIntervalMs: 1000
  backoffMultiplier: 2
  maxIntervalMs: 30000
  retryOn: ["agent_provider_error"]
  nonRetryableErrors: ["agent_needs_human"]
```

Task nodes default to one attempt. Agent `maxRepairAttempts` is separate and
only repairs structured output inside a single agent task attempt.

Agent `task` and `task.each` nodes may declare prompt artifact attachments for
top-level file artifact input fields:

```yaml
promptAttachments:
  imageArtifacts:
    fields: [sourceImage]
  textArtifacts:
    fields: [extractedText]
    maxCharsPerArtifact: 20000
```

Use image attachments for PNG/JPEG/GIF/WebP artifacts and text attachments for
extractor-produced text artifacts.

## Capability

```yaml
apiVersion: kora/v1
kind: Capability
metadata:
  name: triage-pr
spec:
  description: Triage pull request risk.
  agentConfig:
    mode: agentic
    model:
      ref: openai/gpt-5.4-mini
      thinkingLevel: off
    output:
      schemaRef: TriageResult
      maxRepairAttempts: 2
  humanConfig:
    summary: Triage pull request
    form:
      fields:
        - name: approved
          type: boolean
          required: true
```

`agentConfig.model` is optional. Omit it to use the deployment default; when set,
`ref` must be one of the supported hardcoded Pi model refs and the environment
must have a saved key in Settings > Agent models. Exact field support can
change; use `kora schema get capability --json` before relying on a rarely used
field.
