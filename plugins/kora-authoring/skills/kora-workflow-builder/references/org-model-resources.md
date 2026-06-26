# Org model resources

Use `kora schema get <resource> --json` for exact required fields and
enums. Inspect the current source proposal by reading workspace files directly.
This file is the quick reminder for where each resource lives and how it fits
together.

## Source manifest and organization

- `kora.yaml` — source manifest and shared machine-execution defaults
- `org/org.yaml` — organization definition

## Roles, people, agents, assignments

- `org/roles/<name>.yaml` — one role per process lane or ownership slot
- `org/people/<name>.yaml` — one human assignee record
- `org/agents/<name>.yaml` — one AI assignee record
- `org/assignments.yaml` — maps roles to people or agents

For a fresh authored project, create `org/assignments.yaml` even when the
first workflow is service-only, using an empty `spec.roles: {}` registry. Add
role entries only when the workflow has human or agent tasks.

Quick rules:

- roles declare required capabilities
- assignments must reference existing roles and assignees
- assignees must declare the capabilities their assigned roles require
- agent assignments require `aiEligible: true` on the role

## Recommended inspection flow

- read `org/roles/*.yaml`, `org/people/*.yaml`, `org/agents/*.yaml`, and
  `org/assignments.yaml` from the workspace
