---
name: kora-onboarding-guide
description: >
  Read when a user asks what Kora is, how Kora works, how to get started,
  what the main product concepts mean, why Kora exists, or when a first-time
  chat onboarding prompt asks for an intro. This is a user-facing tutorial
  skill, not a workflow-authoring or implementation-detail skill.
---

# Kora Onboarding Guide

Use this skill to explain Kora to a first-time builder in product-facing
language. The goal is to orient the user, teach the mental model, and help them
choose the next specific topic or workflow to explore.

Do not create files, create a release, deploy anything, inspect secrets, or
change workspace state from this skill alone.

Route to a sibling skill when the conversation shifts:

- Building or editing workflow source -> `kora-workflow-builder`.
- Where to click in the product (routes, tabs, menus, settings sections,
  buttons) -> `kora-product-ui`, before naming any of them.
- Authoring an extension package -> `kora-extension-builder`.

## Response Shape

When the user asks for a broad intro, provide a clear first-pass tutorial rather
than trying to exhaust this entire skill in one answer. Prefer clear sections and
plain product language over implementation details.

Default broad-intro shape:

1. Start with Kora's product goal.
2. Explain who Kora is for and the adoption path.
3. Cover the core concepts at overview depth.
4. Explain that workflow building and workflow improvement happen
   conversationally in chat.
5. Explain how change control, events, and governance keep the workflow
   inspectable and controlled.
6. End with follow-up questions the user can choose from.

Do not cover every concept in detail unless the user explicitly asks for a deep
dive. Keep the first answer concise enough to be a good onboarding conversation
opener.

After the explanation, offer focused follow-up questions rather than a long
intake form. Good defaults include:

- Want to walk through a concrete example workflow?
- Want to map an existing process into Kora?
- Want to understand orgs, people, agents, capabilities, and operations?
- Want to see how governance, events, releases, and deployments work?
- Want to build your first workflow in chat?

## Base Intro

Kora is a workflow operating system for important operational business
processes.

It is for teams that already have real workflows: approvals, escalations,
exception handling, quality reviews, fulfillment decisions, maintenance
handoffs, change-control paths, or other processes where execution needs to be
reliable and explainable.

Kora is not a lightweight automation toy and not a chat demo wrapped around
shallow automations. Its product promise is controlled execution. Workflows
should be deterministic where that matters, observable while they run, auditable
after they finish, approval-aware, and integrated with the systems a company
already uses.

A good Kora rollout usually starts with one important workflow. First mirror the
existing human process faithfully. Then make it executable, visible, and
auditable. Once the baseline is trusted, selected steps can move from humans to
agents without throwing away the workflow model.

## Who Kora Is For

Kora is primarily for companies that care about operational reliability and
controlled workflow change.

Typical users include:

- forward deployed engineers or technical implementation leads who understand a
  customer's operational process
- operators who need to see runs, tasks, failures, and approvals
- process owners who care about auditability, handoffs, and controlled change
- builders who refine workflows and gradually increase automation depth

Kora is a good fit when the workflow matters enough that the team needs to know
what changed, what ran, who approved, what failed, and which release was live.

## Adoption Path

Explain the adoption path as a sequence:

1. Mirror the existing workflow.
   Start with the real human process. Preserve the important steps, decisions,
   approvals, vetoes, and escalation paths.
2. Prove reliability.
   Make the workflow executable and observable. Use releases, deployments,
   activity, runs, tasks, and failure evidence to build trust.
3. Introduce agents selectively.
   Move one step at a time from human ownership to agent ownership only where it
   is useful and safe. Keep risky or approval-sensitive steps human-owned.
4. Expand.
   After one workflow works, add more workflows, integrations, and deeper
   automation.

The first successful workflow should usually be narrow enough to model and test
clearly, but important enough that observability, approvals, and auditability
matter.

## Core Concepts

Use this as the concept map. Teach only as much as the user needs in the first
answer, but keep the relationships clear.

| Concept | What it is |
| --- | --- |
| Organization | The Kora workspace/tenant: access members, chat sessions, releases, environments, deployments, runtime configuration, and operational history. |
| Access members | Real humans who log in to Kora. They live in the product access model, not in workflow source; their roles decide what they can do in the workspace. |
| Modeled people | Participants inside a workflow design (reviewer, approver, planner, field technician, escalation or process owner). Not login accounts. |
| Agents | AI assignees in the workflow model. An agent performs a task when a role assignment resolves to it and the capability has agent configuration. |
| Roles | Ownership slots in a process (reviewer, dispatcher, approver, technician, escalation owner, compliance reviewer, purchasing owner). A task references a role. |
| Assignments | Map roles to modeled people or agents -- the link between ownership slots and actual assignees. They let Kora preserve the workflow shape while changing who performs a step. |
| Capabilities | Describe the work a task needs (triage, approval, review, classification, investigation, reconciliation, drafting, exception analysis). Can include human guidance, agent instructions, and output rules. |
| Operations | Service actions for custom code, data transformation, system lookups, notifications, or API calls. A service node calls an operation directly; a task routes work to a person or agent. |
| Extensions | How Kora connects to external systems. Expose functions for operations, tools/skills for agents, settings panels, callbacks, and schedules. Installed into environments; deployment validates the target has what the workflow needs. |
| Processes | The workflow graphs: starts, tasks, service nodes, decisions, gateways, waits, timers, receives, calls to other workflows, and ends. BPMN-inspired, not a strict BPMN runtime -- the goal is to translate real operational process meaning into a reliable executable model. |
| Starts and triggers | A workflow starts from a message, a schedule, or another workflow call; the start defines expected input and where execution begins. For first workflows, frame it in business terms: what event, request, or exception begins the process? |
| Tasks | Units of work assigned to a person or agent: human decisions, agent analysis, structured outputs, approvals, vetoes, or handoffs. Actions are the product surface for pending human tasks. |
| Decisions and gateways | Control routing -- approve, reject, escalate, retry, wait, branch, or end. Make decision points explicit instead of hiding them in unstructured instructions. |
| Source workspace | The proposal area for authored workflow source; may start empty; used for drafting, validation, and safe smoke tests. Not the live runtime -- edits change nothing live until the user creates a release and deploys it. |
| Releases | Immutable snapshots of authored source. Creating one freezes the proposal but makes nothing live. Use them to review, validate, compare, and preserve versions. |
| Environments | Deployment targets and runtime-configuration scopes (e.g. production, staging). Own runtime variables, secret metadata, extension installs, deployment policy, and the live deployment pointer. |
| Deployments | An attempt to apply a release to an environment. Success becomes live; a failed deployment is retained for audit but does not replace the live workflow. Return to an older workflow by deploying that older release again. |
| Runs | Workflow executions. A run shows what started, which steps happened, what state was produced, and what failed, waited, or completed. |
| Activity and events | The operational history across runs, tasks, failures, approvals, deployments, and changes. Part of the product value, not incidental logging. |

Key distinctions to keep clear when teaching:

- The organization (tenant) is not the modeled organization inside a workflow
  release.
- Access members (login accounts, managed in Settings) are not modeled people
  (defined inside a release).
- A task references both a role and a capability; the role assignment decides
  whether the human path or the agent path runs.
- Agents are not the default answer. Make it normal to start with human work,
  move selected steps to agents later, and move them back when needed.

## Conversational Workflow Building

Kora's primary authoring experience is conversational.

A builder can describe a workflow in plain language, paste process notes,
upload existing documentation, or ask Kora to inspect current live state. Kora
then helps turn that conversation into a structured workflow model: people,
agents, roles, assignments, capabilities, operations, decisions, process steps,
releases, and deployment targets.

Useful first messages include:

- "Turn this process into a Kora workflow."
- "Ask me the questions needed to build the first workflow."
- "I will paste process notes. Extract the roles, decisions, approvals, and
  open questions."
- "Start with a mock integration until the real system is connected."
- "Make this approval workflow visible and auditable before we automate it."

Kora should ask focused questions, usually one at a time. The main things to
learn are:

- what starts the workflow
- who participates
- what decisions exist
- what human approvals or vetoes matter
- which systems are involved
- what can fail
- what needs to be visible later
- what should be mocked until a real integration exists
- where the workflow should eventually run

Chat edits are proposals until the user explicitly asks to create a release.
A release freezes the proposal. A deployment makes a release live in an
environment. This keeps conversation fast while preserving controlled change
management.

## Conversational Improvement

Kora is not only for creating the first workflow. Users can improve workflows by
asking for changes in chat.

Examples:

- "Add an approval step before fulfillment."
- "Make this agent step human-owned for now."
- "Explain what changed since the last release."
- "What failed in production?"
- "Where are approvals getting stuck?"
- "Make this workflow safer before we deploy it."
- "Use a mock integration until the real system is connected."
- "Turn these process notes into a first workflow proposal."
- "Compare the live workflow to the release I just created."
- "Draft a safer rollout plan."

As with new workflows, improvements stay proposals until the user creates a
release and deploys it, so iteration stays fast without changing production
behavior.

## Governance, Auditability, And Operational Confidence

Kora is designed for workflows where change control matters.

Important governance surfaces:

- source proposals explain what Kora is drafting
- releases create immutable snapshots
- deployments record what was applied to each environment
- runs show what happened during execution
- tasks show pending human work and completion decisions
- activity and events provide operational history
- failures can be investigated instead of disappearing into logs
- approvals, vetoes, escalations, and agent handoffs stay visible

This model helps teams answer governance questions:

- What release was live in the environment?
- Who or what performed the work?
- Which decision path was taken?
- What input was available at the time?
- What failed, retried, waited, or escalated?
- What changed between the old workflow and the new one?
- Can we safely move this human step to an agent?
- Can we move it back if needed?

The point is not just to automate work. The point is to make important work
executable, observable, reviewable, and improvable.

## Humans And Agents

Explain human and agent ownership as a strength of the product.

Kora should not pressure the user into full autonomy on day one. The safer path
is:

1. model the workflow with human tasks and approvals
2. prove that the workflow runs reliably
3. identify one step where agent help is useful
4. give the agent bounded instructions, tools, and output expectations
5. keep approvals or escalation where risk remains
6. review activity and failures before expanding automation

The same workflow model can support human and agent work. The user should not
need to throw away the workflow to change who owns a step.

## Integrations And External Systems

Kora integrates with external systems through explicit modeled behavior.

Use generic language unless the user names a provider or extension discovery
confirms a provider is available. Say "a system of record",
"messaging channel", "approval system", "ticketing system", or "installed
extension" rather than inventing a provider.

When a real integration is not connected yet, suggest a clearly labeled mock
step with representative inputs and outputs. This lets the user test the
workflow shape before adding credentials or provider setup.

## Good First Use Cases

Good first Kora workflows are important, bounded, and approval-aware.

Examples:

- quality deviation or non-conformance handling
- maintenance or production escalation
- change-control approval
- procurement or fulfillment exception handling
- order exception routing
- field-service escalation
- support escalation with human approval
- compliance review handoff
- incident triage and escalation

The best first workflow usually has:

- a clear trigger
- a known process owner
- one or more human decisions
- a real operational consequence
- a visible success outcome
- enough pain that observability matters
- enough boundaries that it can be modeled safely

## What Not To Do In The Intro

- Do not describe Kora as generic no-code automation.
- Do not imply agents should own the whole workflow immediately.
- Do not blur source proposals, releases, deployments, and live runs.
- Do not say workspace files are already live.
- Do not expose implementation details unless the user asks.
- Do not claim current org state unless you inspected authoritative state.
- Do not name exact UI routes or buttons unless you have read
  `kora-product-ui`.

## Closing Question

End the broad intro with exactly one question:

> What would be most useful to focus on next: a concrete example workflow, the
> core concepts, importing an existing process, or building your first workflow
> in chat?
