# Service IO

Script-backed service nodes invoke an `Operation`.

```yaml
apiVersion: kora/v1
kind: Operation
metadata:
  name: lookup-company
spec:
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
```

The script imports the runtime SDK. For ordinary extension functions, use
`extensions.invoke(alias, functionName, input)`:

```ts
import { extensions, getInput, emitOutput } from "@kora/runtime-sdk";

const input = getInput<{ domain: string }>();
const company = await extensions.invoke("crm", "lookupCompany", {
  domain: input.domain
});

emitOutput({ company });
```

Rules:

- `getInput<T>()` returns the operation input after `paramBindings`.
- `paramBindings` with `from: input` are strict. A binding path that selects a
  field must point through properties required by the service node input schema.
- `extensions.invoke(alias, functionName, input)` calls a registered extension
  function allowed by the operation's runtime SDK binding.
- Cloud-backed built-ins are ordinary extensions from the script perspective.
  Discover their callable functions/tools with `kora extensions search` and
  exact `kora extensions get`, then bind the selected install alias explicitly.
- `spec.bindings.env` injects non-secret runtime variables and
  `spec.bindings.secrets` injects named organization secrets into declared env
  vars for the service script only. These bindings do not grant access to
  extension safe-storage secrets.
- `emitOutput(value)` writes the one JSON value mapped by `resultMapping`.
- Do not print progress logs to stdout when `parseStdoutAsJson` is true; use
  stderr for diagnostics.

For direct extension behavior, invoke the selected function itself. A
token-helper function is only appropriate when that helper is the exact
function contract discovered for the operation.

## Stdout Contract

When `parseStdoutAsJson` is true or omitted, stdout must contain exactly one
valid JSON value. Do not print progress, status, debug text, or banners to
stdout before or after the JSON. Use stderr for logs.

When `parseStdoutAsJson` is false, stdout is treated as raw text. Use that only
when the service node output type and `resultMapping` deliberately expect a raw
string channel.

Bad stdout:

```js
console.log(`Looking up ${domain}`);
console.log(JSON.stringify({ company: null }));
```

## Result Mapping

The service node `output` type should match the operation `resultMapping`
result. Raw script stdout does not automatically become workflow state. If the
script emits `{ "result": { "period": "..." } }`, either map
`$.stdout.result.period` to top-level `period`, or make downstream inputs and
templates use `result.period`.

If `resultMapping` is omitted, parsed stdout must be a JSON object. The runtime
uses that object as the operation result and shallow-merges it into workflow
state according to the service node contract.

Absent and `null` are different runtime values. An optional `resultMapping`
field is omitted only when its source JSONPath is absent. If the source path
exists and contains `null`, that present `null` is mapped as `null` and must
validate against the declared output type.

Omit the property entirely for missing optional values. Do not emit `{}` or
`null` placeholders for a typed success output unless the process type
explicitly allows that shape. If a required value is absent, throw so an error
boundary handles it, return a typed failure/empty-result shape that downstream
nodes understand, or make the specific field optional and handle that branch
deliberately.

If an operation may omit an optional field from stdout, mark that mapping
`optional: true`. Optional mappings are skipped when the source path is absent;
they cannot satisfy required output fields.

## Document And Archive Extraction Helpers

For managed artifact extraction, prefer a small service script that reads
ordinary operation parameters from stdin, reads artifact input and output
`localPath` values from the manifest pointed to by
`process.env.KORA_ARTIFACT_MANIFEST`, calls sandbox tools, writes extracted text
to the manifest-provided declared artifact output path under `/kora/artifacts`,
and emits JSON metadata to stdout.

The default chat and workflow runtime sandbox images include common extraction
CLIs: Poppler tools such as `pdftotext`, `pandoc`, `docx2txt`, `catdoc`,
`catppt`, `w3m`, `unzip`, `zip`, `bsdtar`, `file`, `jq`, and `xmlstarlet`.
These are useful for PDFs, HTML, DOCX, older DOC/PPT best-effort extraction,
PPTX/ZIP XML inspection, and archive handling.
