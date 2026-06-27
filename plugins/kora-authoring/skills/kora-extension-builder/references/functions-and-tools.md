# Functions And Tools

## Contents

- Static functions
- Dynamic function providers
- Static tools
- Dynamic tool providers
- Provider result schemas
- Collision and refresh rules

## Static Functions

Use `registerFunction` for callable backend behavior that workflow service
nodes and agents can call after the extension is installed and granted.

`registerFunction` requires `name`, `description`, `input`, and
`run(input, ctx)`. Optional fields are `output`, `outputPersistence`,
`permission`, and `timeoutMs`. `input` and `output` are TypeBox schemas such as
`Type.Object({ id: Type.String() })`.

```ts
import { Type, type ExtensionHost, type Static } from "@kora/extension-sdk";

const LeadInput = Type.Object(
  {
    company: Type.String({ minLength: 1 }),
    owner: Type.Optional(Type.String())
  },
  { additionalProperties: false }
);
const LeadOutput = Type.Object(
  {
    summary: Type.String()
  },
  { additionalProperties: false }
);
type LeadInput = Static<typeof LeadInput>;

export default function extension(kora: ExtensionHost) {
  kora.registerFunction({
    name: "summarizeLead",
    description: "Summarize a lead record for follow-up.",
    input: LeadInput,
    output: LeadOutput,
    async run(input) {
      const lead = input as LeadInput;
      return {
        summary: lead.owner
          ? `${lead.company} should be followed up by ${lead.owner}.`
          : `${lead.company} should be followed up.`
      };
    }
  });
}
```

## Dynamic Function Providers

Use `registerFunctionProvider` when an install exposes workflow-callable
functions that are not known from package source alone, such as functions
selected from settings, safe storage, secrets, or an external action catalog.

```ts
kora.registerFunctionProvider({
  name: "managed",
  description: "Expose managed actions for this install.",
  permission: "cloud",
  async list(_input, ctx) {
    const catalog = await readManagedCatalog(ctx);
    return {
      functions: catalog.actions.map((action) => ({
        name: action.name,
        description: action.description,
        inputSchema: action.inputSchema,
        outputSchema: action.outputSchema,
        outputPersistence: action.secretOutput ? "ephemeral" : "stored",
        metadata: {
          actionId: action.id,
          schemaVersion: action.schemaVersion
        }
      }))
    };
  },
  async run(input, ctx) {
    if (!ctx.idempotencyKey) {
      throw new Error("Managed action execution requires an idempotency key.");
    }
    return await runManagedAction(ctx, {
      actionId: String(input.metadata?.actionId ?? input.functionName),
      idempotencyKey: ctx.idempotencyKey,
      input: input.input,
      schemaVersion: String(input.metadata?.schemaVersion ?? "")
    });
  }
});
```

The `list` handler returns either an array of functions or
`{ functions: [...] }`. Each listed function supports `name`, `description`,
`inputSchema` or `input`, `outputSchema` or `output`, optional
`outputPersistence`, and optional JSON `metadata`. Platform persists provider
function metadata in the install registration snapshot and passes that pinned
object to the provider `run` handler as `input.metadata`; operation release
preparation also pins the selected provider-backed function registration into
the runtime binding. Use metadata for catalog identity or schema versions that
must not be re-resolved from a live catalog or mutable install registration
snapshot during an already released workflow run. Provider function names share
the same installed workflow function surface as static functions; they must not collide
with static functions or functions from another provider in the same install.

Platform reads cached provider functions from the install registration snapshot.
When a registered provider has no cached provider-owned functions, capability
discovery populates that missing surface once through the provider `list`
handler. Empty `list` results are rejected and are not persisted. Use
`ctx.idempotencyKey` when a provider
function `run` handler performs an external side effect; it is scoped to the
caller-provided idempotency material. If it is absent, fail closed or require an
explicit idempotency key in the function input.

## Static Tools

Use `registerTool` when the installed extension should expose a fixed
agent-runtime tool with a fixed schema.

`registerTool` uses the same core fields as `registerFunction`: `name`,
`description`, `input`, `run(input, ctx)`, and optional `output`,
`outputPersistence`, `permission`, and `timeoutMs`.

```ts
kora.registerTool({
  name: "summarize",
  description: "Summarize selected text for an agent task.",
  input: Type.Object({ text: Type.String() }, { additionalProperties: false }),
  output: Type.Object({ summary: Type.String() }, { additionalProperties: false }),
  async run(input) {
    return { summary: `Summary: ${input.text}` };
  }
});
```

## Dynamic Tool Providers

Use `registerToolProvider` when an install exposes agent tools that are not
known from package source alone, such as tools discovered from safe storage,
settings, secrets, or an external catalog.

```ts
kora.registerToolProvider({
  name: "remote",
  description: "Expose tools discovered for this install.",
  permission: "storage",
  async list(_input, ctx) {
    const configured = await ctx.storage.get("tool-config");
    return {
      tools: configured.tools.map((tool) => ({
        name: tool.name,
        description: tool.description,
        inputSchema: tool.inputSchema,
        outputSchema: tool.outputSchema,
        outputPersistence: tool.secretOutput ? "ephemeral" : "stored"
      }))
    };
  },
  async run(input, ctx) {
    return await callRemoteTool(ctx, input.toolName, input.input);
  }
});
```

The `list` handler returns either an array of tools or `{ tools: [...] }`.
Each listed tool supports `name`, `description`, `inputSchema` or `input`,
`outputSchema` or `output`, and optional `outputPersistence`. Provider tool
names share the same installed agent surface as static tools; they must not
collide with static tools or tools from another provider in the same install.

Platform reads cached provider tools from the install registration snapshot.
When a registered provider has no cached provider-owned tools, capability
discovery populates that missing surface once through the provider `list`
handler. Empty `list` results are rejected and are not persisted.

Provider functions are for workflow/service-node calls. Provider tools are for
agent tool calls. If the same remote action should be available to both,
register both surfaces deliberately so each can have the right description and
schema for its caller.
