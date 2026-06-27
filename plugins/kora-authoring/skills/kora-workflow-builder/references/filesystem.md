# Runtime Filesystem And Dependencies

The chat source workspace is not the live runtime. Use it for source editing
and smoke tests only.

Service operation scripts run in OpenSandbox with released deployment files
mounted read-only at `/workspace`. Runtime code can read packaged release files,
but must not write generated reports, caches, logs, lockfiles, temp files, or
mutable state next to deployed source files.

Files written during a run are scratch state for that run and are discarded
before future runs. Use `/tmp` or Kora runtime output paths for scratch files.
Declare file/folder outputs when downstream workflow steps need durable managed
artifacts. Do not design runtime memory around local state files.

Extension package code runs through Extension Host in its own sandbox
invocation. Operation scripts access extension behavior through
`@kora/runtime-sdk`, not by reading extension source files from the workflow
workspace.

## `kora` Boundary

`kora` is an authoring and inspection tool. It is not available inside live
workflows, service operations, scripts, or agent tasks. Do not put `kora`
commands in workflow YAML, scripts, operations, or agent prompts.

## Dependency Availability

Package-manager installs performed while authoring are scratch state. Do not
rely on installed `node_modules`, virtualenvs, caches, or generated build
output as deployable project state.

Live runs use a separate sandbox and release artifact. `node_modules` is not
packaged as live project state. Commands and scripts must run with files in the
release artifact plus dependencies available in the configured runtime image.

The default chat and workflow runtime sandbox images include a small set of
document/archive extraction CLIs that service scripts can call when appropriate:

- `pdftotext`, `pdfinfo`, and `pdftohtml` from Poppler for digital PDFs
- `pandoc` for DOCX/HTML/Markdown-oriented text conversion
- `docx2txt` for simple DOCX text extraction
- `catdoc` and `catppt` for older binary DOC/PPT best-effort extraction
- `w3m` for HTML-to-text dumps
- `unzip`, `zip`, and `bsdtar` for ZIP/archive inspection and extraction
- `file`, `jq`, and `xmlstarlet` for MIME sniffing, JSON, and XML processing

These are command-line tools, not Node package imports. Do not write service
scripts that import npm packages such as `pdfjs-dist` or `mammoth` unless the
project has a deterministic packaging strategy for those dependencies.

Avoid `npm install`, `pip install`, or similar package-manager installs as part
of normal operations unless the org has explicitly approved a deterministic
runtime/dependency strategy for it.
