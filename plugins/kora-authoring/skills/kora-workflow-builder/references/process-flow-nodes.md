# Process Flow Nodes

Processes are BPMN-inspired YAML definitions. State is one top-level object.
Each node output shallow-merges into state.

## Starts

```yaml
start:
  - type: message
    name: external-event
    input: EventInput
    goto: first-node
  - type: timer
    schedule: daily
    goto: first-node
  - type: manual
    internal: true
    input: StartInput
    goto: first-node
```

Use `message` for external inbound triggers and `timer` for scheduled
top-level workflows. A process called by another process should expose a
manual start with `internal: true`. Bare top-level manual starts are invalid.

## Service

Use service nodes for custom code and external behavior.

```yaml
- id: read-prs
  type: service
  operation: read-github-prs
  input: PullRequestQuery
  output: PullRequestBatch
  next: review
```

The `operation` resource runs a script. That script can call installed
extension functions through `@kora/runtime-sdk`.

Service nodes do not require `role`, `capability`, or assignment resources.
Add org-model resources only for human/agent task nodes.

## Human Or Agent Task

```yaml
- id: review
  type: task
  role: reviewer
  capability: triage-pr
  input: PullRequestBatch
  output: TriageResult
  promptAttachments:
    imageArtifacts:
      fields: [sourceImage]
    textArtifacts:
      fields: [extractedText]
      maxCharsPerArtifact: 20000
  retry:
    maxAttempts: 2
    retryOn: [agent_provider_error]
    nonRetryableErrors: [agent_needs_human]
  next: done
```

The role assignment decides whether a person or an agent performs the task.
Task retry is whole-node retry. Tasks default to one attempt; use `retry` only
for failures that are safe to run again. Agent structured-output repair remains
configured on `agentConfig.output.maxRepairAttempts` and does not retry the
whole task node.

`promptAttachments` applies only to agent tasks. The `fields` entries are
top-level `x-kora-type: file` fields from the task input type, not artifact IDs
or JSON pointers. Use it for selected image artifacts or extractor-produced text
artifacts that should be included in the initial remote model prompt.

## Collections

Use `task.each` or `call.each` for runtime collections.

```yaml
- id: review-each
  type: task.each
  role: reviewer
  capability: triage-pr
  collection: $.pullRequests
  itemInput: PullRequest
  itemOutput: TriageResult
  promptAttachments:
    textArtifacts:
      fields: [extractedText]
  retry:
    maxAttempts: 2
    retryOn: [agent_provider_error]
  inputMapping:
    pullRequest:
      from: $.item
  outputMapping:
    reviews:
      from: $.results
  next: done
```

For `task.each`, prompt attachment fields refer to top-level fields on
`itemInput` after `inputMapping`.

## Decisions And Gateways

Use `decision` for reusable decision logic and gateways for inline routing.

```yaml
- id: choose-path
  type: gateway.exclusive
  paths:
    - condition: approved == true
      goto: done
    - default: true
      goto: needs-work
```

## Receive And Timers

`receive` waits for a named message. `timer` waits for a duration.

```yaml
- id: wait-for-webhook
  type: receive
  catch: external-update
  output: ExternalUpdate
  next: done
```

External systems should enter through Platform-owned ingress, including
extension callbacks when the integration is extension-owned.
