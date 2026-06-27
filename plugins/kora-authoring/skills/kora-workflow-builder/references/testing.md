# Testing Runtime Assets

Use the cheapest safe check that exercises the actual contract.

1. For pure transformation scripts, run the script locally with representative
   JSON input and verify `emitOutput` produces the expected JSON.
2. For script-backed service nodes, use
   `kora test node <workflow-name> <node-id> --workspace <dir> --input @<dir>/input.json --environment <environment> --json`
   with representative input, or run from the workspace directory and use
   `--workspace . --input @input.json --environment <environment>`. This includes
   service nodes whose operations call installed extension functions through
   `@kora/runtime-sdk`.
3. For artifact outputs, use a workflow-node test or a real workflow run. Do
   not claim release creation, release validation, or deployment proves artifact
   capture.

A failed workflow-node test is a publish blocker. Do not create a release,
deploy, or tell the user the workflow is ready unless the test passes or the
user explicitly approves publishing anyway after you warn it is likely to fail.
Classify failures before responding:

- source/validation error
- missing extension/binding/grant
- missing connection/credential
- extension runtime error
- provider/API error

If `kora test node ... --json` reports missing runtime SDK extension support, call
that a Platform test-harness gap. Do not claim extension-backed service nodes are
untestable in the current authoring environment.

Do not claim readiness if a script was only typechecked but never executed with
representative input.
