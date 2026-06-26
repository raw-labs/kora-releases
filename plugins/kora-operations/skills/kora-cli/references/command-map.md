# Command Map

Top-level command families. Use `kora help <command-path> --json` for exact
subcommands, arguments, and flags; this map only routes you to the right
family.

- `kora auth` — sign up, log in, log out, and inspect the current session
- `kora org` — create, list, select, and manage organizations
- `kora status` — operator home/status summary for the active organization
- `kora activity` — activity events with related tasks and runs
- `kora workflow` — list live deployed workflows by default, filter that list
  with `--environment <environment>`, inspect workflow detail/context/dependencies
  with `--release <release>` or `--environment <environment>`, and start workflows
- `kora test` — test one workflow node from source files without a release
- `kora release` — create and inspect immutable releases
- `kora run` — workflow runs: list, inspect, control
- `kora audit` — organization audit events
- `kora artifact` — upload and download managed runtime artifacts
- `kora task` — human tasks and the inbox
- `kora deployment` — environment deployments
- `kora environment` — environments and deploying releases to them
- `kora org-model` — release-scoped modeled people, agents, roles,
  assignments, capabilities, and operations; reads require `--release <release>`
  or `--environment <environment>`
- `kora access` — members, invites, and organization API keys
- `kora extensions` — validate, publish, install, grant, and manage
  extension installs
- `kora secrets` — organization secrets (list shows names without values)
- `kora env` — runtime variables
- `kora chat` — chat health and helpers
- `kora schema` — resource schemas exposed by the CLI
- `kora admin` — platform-admin reads such as usage summaries
- `kora completion` — shell completion
- `kora help` — human or machine-readable help for a command path

Aliases:

- `kora home` → `kora status`
- `kora inbox` → `kora task`
- `kora process` → `kora workflow`
- `kora project` → `kora org`

Unlike the in-product chat agent, the CLI surface is not label-filtered:
extension install and permission grants (`kora extensions install`,
`kora extensions grant`) are available here, while chat intentionally blocks
them and routes users to Settings.
