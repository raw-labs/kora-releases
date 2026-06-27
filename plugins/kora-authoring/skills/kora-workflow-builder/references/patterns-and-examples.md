# Patterns And Examples

## Complete Minimal Service Workflow

Use this as the approved starting point for a fresh service-only workflow. It
has no human or agent tasks, so `org/assignments.yaml` keeps an empty
`spec.roles` registry.

```text
kora.yaml
org/org.yaml
org/assignments.yaml
operations/generate-random-number.yaml
processes/generate-random-number.yaml
scripts/generate-random-number.mjs
```

`kora.yaml`

```yaml
apiVersion: kora/v1
kind: Project
metadata:
  name: random-number-project
  description: Generate random numbers
```

`org/org.yaml`

```yaml
apiVersion: kora/v1
kind: Organization
metadata:
  name: random-number-org
spec:
  teams: []
```

`org/assignments.yaml`

```yaml
apiVersion: kora/v1
kind: Assignments
metadata:
  name: random-number-assignments
  version: 1
spec:
  roles: {}
```

`operations/generate-random-number.yaml`

```yaml
apiVersion: kora/v1
kind: Operation
metadata:
  name: generate-random-number
spec:
  description: Generate a random integer from 1 to 100
  script:
    command: node
    args:
      - scripts/generate-random-number.mjs
  resultMapping:
    randomNumber:
      from: $.stdout.randomNumber
```

`processes/generate-random-number.yaml`

```yaml
apiVersion: kora/v1
kind: Process
metadata:
  name: generate-random-number
  version: 1
  description: Generate a random number
types:
  EmptyInput:
    type: object
    properties: {}
  RandomNumberOutput:
    type: object
    properties:
      randomNumber:
        type: integer
        minimum: 1
        maximum: 100
    required: [randomNumber]
start:
  - type: message
    name: generate-random-number
    input: EmptyInput
    goto: generate
flow:
  - id: generate
    type: service
    operation: generate-random-number
    input: EmptyInput
    output: RandomNumberOutput
    next: done
  - id: done
    type: none
```

`scripts/generate-random-number.mjs`

```js
import { emitOutput } from "@kora/runtime-sdk";

const randomNumber = Math.floor(Math.random() * 100) + 1;

emitOutput({ randomNumber });
```

## Human Review After Service Work

```yaml
flow:
  - id: read-prs
    type: service
    operation: read-github-prs
    input: PullRequestQuery
    output: PullRequestBatch
    next: review
  - id: review
    type: task
    role: reviewer
    capability: triage-pr
    input: PullRequestBatch
    output: TriageResult
    next: done
  - id: done
    type: none
```

The service script may call an extension function to get provider-specific
behavior. The process does not model the provider directly.

## Extension-Backed Operation

```text
operations/read-github-prs.yaml
scripts/read-github-prs.ts
processes/pr-review.yaml
```

Extension source is not part of workflow release source. Publish and
install the extension through the extension lifecycle first, then make
`operations/read-github-prs.yaml` grant the script a runtime SDK alias. The
script imports `@kora/runtime-sdk`, calls the exact selected extension function
with `extensions.invoke("github", "GITHUB_LIST_PULL_REQUESTS", input)`, and
emits JSON with `emitOutput`.

## Empty Result Branch

Model expected absence explicitly.

```yaml
- id: route-empty
  type: gateway.exclusive
  paths:
    - condition: size(pullRequests) == 0
      goto: done
    - default: true
      goto: review
```

Avoid returning placeholder `null` values unless the declared type allows
`null`.
