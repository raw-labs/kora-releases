# Managed Artifacts

Managed workflow artifacts are platform-managed file or folder values. They are
not chat source workspace files, object-store URLs, storage keys, or signed
storage URLs.

During an authoring session, uploaded files start as unclassified workspace files,
not managed artifacts. If the user explicitly asks to use a selected workspace
file as a workflow/run artifact input, upload it with
`kora artifact upload <path> --json` and use the returned artifact ref.
Route artifact inspection and downloads to the Platform run/task artifact UI.
The workspace may author artifact-aware workflows, but it must not silently
turn local workspace files into live workflow artifacts.

## Type Declarations

Declare top-level file or folder artifact fields in process types with
`x-kora-type: file` or `x-kora-type: folder`.

```yaml
types:
  ArtifactPackageInput:
    type: object
    properties:
      sourceData:
        type: object
        x-kora-type: file
    required: [sourceData]

  ArtifactPackageOutput:
    type: object
    properties:
      reportPdf:
        type: object
        x-kora-type: file
      assetFolder:
        type: object
        x-kora-type: folder
    required: [reportPdf, assetFolder]
```

Artifact declarations are top-level object fields in the process type. Do not
model nested artifact declarations, arrays of artifacts, or artifact
declarations under `additionalProperties`.

## Runtime Input Contract

When a CLI service node has declared file or folder artifact input fields, Core
stages those artifact bytes into the service workspace before command execution.
If stdin is JSON object data containing declared input artifact fields, those
fields are enriched with execution-local `localPath` values.

Agent task nodes use the same declared-artifact input contract. The agent input
JSON receives execution-local `localPath` values, and the prompt includes the
artifact manifest path when an artifact workspace exists. A `task` or
`task.each` node can opt selected top-level file artifact input fields into the
initial model prompt:

```yaml
promptAttachments:
  imageArtifacts:
    fields: [attachmentImage]
  textArtifacts:
    fields: [attachmentText]
    maxCharsPerArtifact: 20000
```

For supported image file inputs (`image/png`, `image/jpeg`, `image/gif`, or
`image/webp`), the Pi engine attaches the staged image to the initial model
prompt only when the selected model advertises image input support. For text
artifact fields, the engine reads the staged local file and includes bounded
text in the initial prompt. The artifact ref remains the workflow value; prompt
files are execution-time convenience only and are not available for
folders, PDFs, Office docs, ZIPs, oversized images, binary-looking text files,
non-allowlisted fields, or models without the needed input support.

Use an extractor service node when an incoming attachment is PDF, DOCX, PPTX,
HTML, ZIP, or another rich/binary format. That node should create a new managed
text artifact, or an image artifact when the downstream agent needs vision, and
the later agent task should attach that extracted artifact field.

```js
const input = await readJsonFromStdin();
const sourcePath = input.sourceData?.localPath;
if (!sourcePath) {
  throw new Error("sourceData.localPath was not staged.");
}
```

`localPath` is execution-local only. It must not be written into durable
workflow state, templates, storage records, or user-facing artifact refs.

## Artifact Manifest Contract

If you find yourself writing a declared output to `/tmp` or `/kora/output`, stop; those paths are for unmanaged scratch files only. Declared outputs must use the manifest's `localPath`.

For service operations with declared file or folder artifact fields, runtime
sets `KORA_ARTIFACT_MANIFEST` to a JSON manifest path. The manifest has `inputs`
and `outputs` arrays with entries keyed by JSON Pointer `fieldPath` and
execution-local `localPath`.

When authoring an artifact-aware service script:

- Read `KORA_ARTIFACT_MANIFEST` and parse the manifest JSON.
- Find inputs and outputs by their declared `fieldPath`.
- Read artifact inputs only from the manifest entry `localPath`.
- Write each declared artifact output only under its manifest entry `localPath`.
- Do not derive artifact paths from release source files, support file paths,
  `/workspace`, `/tmp`, `/kora/output`, storage keys, URLs, or host filesystem
  paths.

The script-visible artifact paths are under `/kora/artifacts`, but the manifest
values are the source of truth. Do not construct subpaths by hand; copy or join
only below the manifest-provided `localPath` for that field.

Shape example; use the manifest values as given and do not depend on the exact
directory names:

```json
{
  "inputs": [
    { "fieldPath": "/sourceData", "localPath": "/kora/artifacts/runs/run_123/nodes/read-prs/activities/activity_1/input/sourceData/input.pdf" }
  ],
  "outputs": [
    { "fieldPath": "/reportPdf", "localPath": "/kora/artifacts/runs/run_123/nodes/read-prs/activities/activity_1/output/attempt-1/reportPdf" },
    { "fieldPath": "/assetFolder", "localPath": "/kora/artifacts/runs/run_123/nodes/read-prs/activities/activity_1/output/attempt-1/assetFolder" }
  ]
}
```

Scripts read the manifest and write declared outputs into the matching
manifest-provided output `localPath`. For file outputs, create the output
directory and write the file inside it. For folder outputs, create the folder
tree under that output path.

```js
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { join } from "node:path";

const manifestPath = process.env.KORA_ARTIFACT_MANIFEST;
if (!manifestPath) {
  throw new Error("KORA_ARTIFACT_MANIFEST is required for managed artifact output.");
}

const manifest = JSON.parse(await readFile(manifestPath, "utf8"));

function findOutputPath(fieldPath) {
  const output = manifest.outputs?.find((entry) => entry.fieldPath === fieldPath);
  if (!output?.localPath) {
    throw new Error(`Artifact output ${fieldPath} was not declared in the manifest.`);
  }
  return output.localPath;
}

const reportRoot = findOutputPath("/reportPdf");
await mkdir(reportRoot, { recursive: true });
const pdfBytes = Buffer.from("%PDF-1.4\n% generated report\n");
await writeFile(join(reportRoot, "report.pdf"), pdfBytes);
```

Runtime captures declared outputs from the manifest-provided paths after the
service script returns. Stdout and `resultMapping` stay the structured JSON
result channel; host-captured refs are injected into declared artifact output
fields after mapping and win for those fields.

## Send Templates

If the send input includes managed artifact refs, the live runtime can add a
transient `url` property while rendering the template when the Platform artifact
link provider is configured. A template may use `{{reportPdf.url}}` or
`{{assetFolder.url}}` for an artifact field declared as `x-kora-type:
file|folder`.

That `url` is a transient delivery value. Do not write artifact URLs into
durable workflow state.
