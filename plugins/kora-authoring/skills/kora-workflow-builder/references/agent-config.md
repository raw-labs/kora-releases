# Agent Config

Agent config lives on a `Capability` under `spec.agentConfig`. It controls how
an agent-assigned task is executed when a role assignment resolves to an agent.

```yaml
spec:
  agentConfig:
    mode: agentic
    systemPrompt: |
      Review the input and return the declared output JSON.
    model:
      ref: openai/gpt-5.4-mini
      thinkingLevel: off
    limits:
      maxTurns: 8
      maxDurationMs: 600000
      maxBudgetUsd: 2
    extensions:
      - github-primary
    execution:
      sandbox:
        network:
          defaultAction: deny
          inheritManaged: false
    output:
      schemaRef: ReviewResult
      maxRepairAttempts: 2
```

Rules:

- Use `mode: prompt` for single-turn prompt execution.
- Use `mode: agentic` when the task needs iterative tool use.
- Omit `agentConfig.model` to use the deployment default agent model chosen at
  install/configure time.
- Use `agentConfig.model.ref` only for supported hardcoded Pi model refs. Do not
  invent arbitrary provider/model strings.
- `agentConfig.model.thinkingLevel` is optional. Use `off` to suppress thinking
  for that capability; otherwise use one of the supported runtime levels.
- Environment owners must save the selected model's API key under Settings >
  Agent models before a release using that explicit model can deploy. Model API
  keys are not stored in YAML.
- Built-in agent tools are provided by the runtime for `mode: agentic`.
- External APIs and domain-specific behavior should be modeled as extension
  functions, extension agent surfaces, and service operations, not hidden in
  the agent config.
- `agentConfig.extensions` is only for agent task nodes. It does not expose
  tools or skills to the workflow-authoring environment.
- Each `agentConfig.extensions` entry names an installed extension. Enabling it
  with shorthand exposes all granted static tools, provider-discovered tools,
  and skills from that install.
- Use object form to restrict an agent task to selected extension tools and
  skills:

  ```yaml
  extensions:
    - name: github-primary
      tools:
        - searchIssues
        - createIssue
      skills:
        - github-triage
  ```

- In object form, `tools` and `skills` may be `all`; omitted categories mean
  none, and the entry must include at least one of `tools` or `skills`.
- Extension skills reference registered skill roots in the installed package
  revision; the root must contain `SKILL.md` and may include adjacent assets.
- Agent output should match the process node's declared `output` type.
- `output.maxRepairAttempts` repairs structured output inside one agent task
  attempt. Whole-node retry belongs on the process `task` or `task.each` node,
  not in `agentConfig`.
- Prompt artifact attachments are process-node settings, not capability
  settings. Put `promptAttachments` on the specific `task` or `task.each` node
  that should send selected top-level file artifact input fields to the model.
- Use image attachments only for actual PNG/JPEG/GIF/WebP artifacts and text
  attachments only for extractor-produced text artifacts. For PDFs, Office
  docs, HTML, ZIPs, or other rich/binary files, model an extractor service node
  first and attach the extracted text or image artifact.

## Supported Model Refs

Keep this list aligned with the Core supported agent model registry and
`koractl` model choices:

<!-- supported-agent-model-refs:start -->
- `anthropic/claude-opus-4-6`
- `anthropic/claude-opus-4-8`
- `anthropic/claude-opus-4-7`
- `anthropic/claude-sonnet-4-6`
- `anthropic/claude-haiku-4-5`
- `openai/gpt-5.5`
- `openai/gpt-5.4`
- `openai/gpt-5.4-mini`
- `google/gemini-3.5-flash`
- `google/gemini-3.1-pro-preview`
- `xai/grok-4.3`
- `moonshotai/kimi-k2.6`
- `deepseek/deepseek-v4-pro`
- `deepseek/deepseek-v4-flash`
- `zai/glm-5.1`
- `zai/glm-4.7`
- `minimax/MiniMax-M3`
- `mistral/mistral-medium-3.5`
- `mistral/mistral-large-2512`
- `groq/meta-llama/llama-4-scout-17b-16e-instruct`
- `openrouter/meta-llama/llama-4-scout`
- `openrouter/qwen/qwen3.7-max`
- `openrouter/qwen/qwen3.6-35b-a3b`
- `openrouter/qwen/qwen3.6-27b`
- `google/gemma-4-31b-it`
<!-- supported-agent-model-refs:end -->
