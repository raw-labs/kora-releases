# Capability Resource

Capabilities describe executable work that process task nodes reference. A role
can require a capability, and assignments decide whether a human or agent
performs it.

```yaml
apiVersion: kora/v1
kind: Capability
metadata:
  name: assess-change
  category: engineering
spec:
  description: Review pull request changes and produce a decision.
  agentConfig:
    mode: agentic
    systemPrompt: |
      Review the pull request data and return the declared output.
    output:
      schemaRef: ReviewDecision
      maxRepairAttempts: 2
  humanConfig:
    summary: Review pull request
    guide: Check risk, tests, and deployment impact.
    priority: normal
    form:
      fields:
        - name: approved
          type: boolean
          required: true
```

Rules:

- `agentConfig` is used when the assigned role resolves to an agent.
- `humanConfig` is used when the assigned role resolves to a person.
- Omit `agentConfig.model` to use the deployment default agent model. If a
  capability needs a different supported model, set `agentConfig.model.ref` and
  ensure the target environment has that model key saved in Settings > Agent
  models.
- Use `agentConfig.extensions` to enable installed extension tools and skills
  for agent tasks; use object-form selections when the capability should expose
  only specific registered tools or skills.
- Human task routing is Platform/Core task state. Notifications or external
  delivery should be modeled as service operations when needed.
- Keep capability descriptions focused on the work, not on provider plumbing.
