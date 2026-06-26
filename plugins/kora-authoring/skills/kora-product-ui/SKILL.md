---
name: kora-product-ui
description: >
  Use when routing a user to the Platform web UI or naming current app
  navigation, routes, settings sections, tabs, buttons, or product surfaces.
---

# Kora Product UI

Use this skill before naming a Kora web-app destination. Do not invent
settings, menus, tabs, buttons, or routes. If a surface is not listed here,
say you do not see a current web UI destination for it.

This skill owns product destinations and visible UI labels. It does not own
workflow source authoring, release/deployment command choreography, runtime IO
contracts, or extension package authoring.

## Link Rules

- Use relative Markdown links for Platform UI routes, such as
  `[release detail](/app/releases/<releaseId>)`.
- Do not format UI routes as code when they should be clickable.
- Include query parameters only when they select a real section, tab, focus, or
  environment.
- Route managed runtime artifact inspection and downloads through Platform UI
  surfaces. Do not point users to object-store buckets, storage keys, filesystem
  paths, or signed storage URLs.

## Shell

The app lives under `/app` and uses a left rail.

| Surface | Route | What it owns |
| --- | --- | --- |
| Home | `/app` | Workspace overview, recent health, deployments, runs, events, and links into Releases, Activity, and Environments. |
| Actions | `/app/inbox` | Human work queue and action completion. |
| Deployed workflows | `/app/processes` | Live workflows currently deployed to environments. |
| Releases | `/app/releases` | Immutable release artifacts, release inventory, source import, and deployment targets. |
| Environments | `/app/environments` | Environment inventory, detail, live deployment status, and environment lifecycle. |
| Activity | `/app/activity` | Runs, events, failures, and focused activity by person, agent, capability, or workflow. |
| Settings | `/app/settings?section=workspace` | Workspace, access, extensions, variables, secrets, agent models, artifacts, audit, usage, plan, and reset sections. |

Workspace and account switching live at the bottom of the rail. Profile is
`/app/profile`, not a Settings section.

## Home

Use `/app` for broad workspace orientation. The page summarizes release and
deployment health, recent runs/events, workspace participants, and quick links
to release inventory, runtime activity, and deployment targets.

## Actions

| Surface | Route | Visible behavior |
| --- | --- | --- |
| Actions list | `/app/inbox` | Pending human actions with owner, capability, due/release, and context columns. |
| Action detail | `/app/tasks/:taskId` | Action overview, guidance, input data, completion form, history, `Open run`, and sometimes `Abort run`. |

Use Actions for human tasks awaiting input or completion. Use Activity for run
history and runtime events.

## Deployed Workflows

| Surface | Route | Visible behavior |
| --- | --- | --- |
| Deployed workflow list | `/app/processes` | Live deployed workflows by environment, release, last deploy time, last run, and run state. |
| Environment filter | `/app/processes?environment=<environmentKey>` | Narrows the deployed workflow table to one environment. |
| Live workflow detail | `/app/workflows/:name?environment=<environmentKey>&releaseId=<releaseId>` | Live workflow graph and release-backed workflow metadata. |

Workflow rows open the live workflow detail. The list and detail surfaces can
start a workflow when a live environment and release are available. Starting a
workflow opens a `Start workflow` dialog; successful starts navigate to
`/app/activity/runs/:workflowId` with workflow and environment focus.

## Releases

Releases are top-level. They are not under Deployed workflows or Settings.
Creating a release produces an immutable artifact from source. Deploying that
release to an environment is a separate action.

| Surface | Route | Visible behavior |
| --- | --- | --- |
| Release list | `/app/releases` | Release revisions, creation metadata, validation/readiness status, action menu, and source import. |
| Release detail | `/app/releases/:releaseId` | Defaults to Workflows. The left rail switches to release sections. |
| Release workflow detail | `/app/releases/:releaseId?section=workflows&workflow=<workflowName>` | Release workflow graph, source files, assets, and `Start workflow` when deployable context exists. |
| Release item detail | `/app/releases/:releaseId?section=<section>&item=<itemName>` | Detail for modeled people, agents, roles, assignments, capabilities, operations, or decisions. |
| Release deployments | `/app/releases/:releaseId?section=deployments` | Current deployments and deployment targets for the release. |

Release sections:

| Group | Sections |
| --- | --- |
| Release | Workflows |
| Organization | People, Agents, Roles, Assignments, Capabilities, Operations, Decisions |
| Runtime | Deployments |

Visible release actions include `Import release`, `Create release from source`
inside the import dialog, opening a release detail, `Deploy to environment...`
from the release list, `Deploy this release...` or `Replace with this
release...` from release deployment targets, the deploy dialog's `Deploy to
environment` confirmation, starting a workflow from a release workflow, and
returning to an older release by deploying that release again.

## Environments

| Surface | Route | Visible behavior |
| --- | --- | --- |
| Environment list | `/app/environments` | Environment inventory and `Add environment`. |
| Environment detail | `/app/environments/:environmentKey` | Environment metadata, deployment status, links to release and deployed workflows, and undeploy controls. |

Visible actions include `Add environment`, `View`, `View workflows` when live,
`Rename`, `Archive`, and `Undeploy` from a live deployment row on the
environment detail page. Environment management is owner/admin gated.

Use Environments for environment lifecycle and live deployment status. Use
Settings -> `Environment variables` for runtime configuration values.

## Activity

| Surface | Route | Visible behavior |
| --- | --- | --- |
| Runs tab | `/app/activity` | Default tab. Workflow execution history by environment and release. |
| Events tab | `/app/activity?tab=all` | Runtime events. |
| People tab | `/app/activity?tab=people` | Focus activity for one modeled person. |
| Agents tab | `/app/activity?tab=agents` | Focus activity for one modeled agent. |
| Capabilities tab | `/app/activity?tab=capabilities` | Focus activity for one capability. |
| Workflows tab | `/app/activity?tab=workflows` | Focus activity for one workflow. |
| Failures tab | `/app/activity?tab=failures` | Failed runs and events needing review. |
| Activity run detail | `/app/activity/runs/:workflowId` | Execution graph, timeline, step details, run metadata, artifacts, and abort controls when allowed. |
| Run detail from Actions | `/app/runs/:workflowId` | Same run detail view. The action detail `Open run` button uses this path. |

Activity supports query filters such as `environment`, `range`, `focusType`,
`focusValue`, run status/search/sort/pagination filters, and task filters.
When linking, include only the filters needed to land the user in the right
context.

Run detail links back to release, release workflow, environment, and release
deployment surfaces. Use the run detail page for runtime event timelines,
execution graph inspection, step-level details, and run artifacts.

## Settings

Settings is the owner for workspace and instance configuration. The canonical
general route is `/app/settings?section=workspace`.

| Group | Section | Route | What it owns |
| --- | --- | --- | --- |
| Workspace | General | `/app/settings?section=workspace` | Workspace/org display settings. |
| Workspace | Members & access | `/app/settings?section=members` | Human members, invites, role changes, invite links, and removals. |
| Workspace | API keys | `/app/settings?section=api-keys` | Organization-scoped API credentials. New tokens are shown only once. |
| Workspace | Extensions | `/app/settings?section=extensions` | Installed extension packages, setup, management, permissions, schedules, and package install flows. |
| Workspace | Environment variables | `/app/settings?section=environment-variables` | Runtime variable inventory, create/edit/delete, and environment filters. |
| Workspace | Secrets | `/app/settings?section=secrets` | Org and environment-scoped secret storage, create/edit/delete, and environment filters. |
| Workspace | Agent models | `/app/settings?section=agent-models` | Environment-scoped agent model API keys for supported model refs. Values are write-only; delete is blocked when a live deployment depends on the model. |
| Workspace | Artifacts | `/app/settings?section=artifacts` | Runtime artifact inventory, upload, preview/detail, download, archive, restore, and purge. Owner/admin gated. |
| Workspace | Audit | `/app/settings?section=audit` | Audit/event review, filtering, pagination, and event detail. Owner/admin gated. |
| Instance | Usage | `/app/settings?section=usage` | LLM usage summary, workflow/tool-call/recent-execution usage. Platform-admin gated. |
| Instance | Plan | `/app/settings?section=license` | Plan/license status and account management link. Platform-admin gated. |
| Danger zone | Reset | `/app/settings?section=danger` | Owner-only `Reset workspace project`. Clears release history, source storage, environment-scoped configuration, and workspaces. |
| Account | Profile | `/app/profile` | User account profile. Not an embedded Settings section. |

Settings actions:

- Members & access: `Invite`, role changes for non-owner members, `Remove`,
  `Copy link` for created invites, and `Cancel` for pending invites.
- API keys: `Create key`, `Copy`, and `Revoke`.
- Extensions: `Add extension`, `Refresh`, setup, manage, enable/disable,
  delete install, save permissions, run declared settings buttons, and open
  external action windows.
- Environment variables: `Add variable`, edit, delete, search/filter, and
  environment selection in create/edit dialogs.
- Secrets: `Add secret`, edit, delete, search/filter, and environment selection
  in create/edit dialogs.
- Artifacts: upload, refresh, preview/detail, download, archive, restore,
  purge, and pagination.
- Audit: filtering by category/source/action/resource and selecting an event
  for detail.

Do not confuse access members with release-modeled people. Access members live
in Settings -> `Members & access`; modeled people live inside a release detail.

## Extension Setup Links

For any extension install, setup, connection, permission, schedule, or
management issue, use this exact Markdown link:

```md
[install and configure the extension](/app/settings?section=extensions)
```

## Artifact Surfaces

Use Platform UI routes for managed runtime artifacts:

- Run artifacts: `/app/activity/runs/:workflowId`, in the run detail artifact
  and step/detail surfaces.
- Task input artifacts: `/app/tasks/:taskId`, in the action detail input
  section.
- Organization artifact inventory: `/app/settings?section=artifacts`.

Start-workflow dialogs can upload supported `x-kora-type: file` inputs before
starting a run. Runtime artifact references in task input data render as
artifact cards. Link to the appropriate UI surface, not to raw object storage.
