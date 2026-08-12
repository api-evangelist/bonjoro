---
name: Automate Bonjoro greet creation from a connected service
description: Inspect and manage Bonjoro automations — the trigger/action rules that create video tasks from a connected CRM or email platform — over the API V2 automation surface.
api: openapi/bonjoro-api-v2-openapi.yml
base_url: https://www.bonjoro.com/api/v2
operations:
  - getAutomationServices
  - getAutomationAccounts
  - testAutomationAccount
  - getAutomationTriggers
  - postAutomationTrigger
  - putAutomationTrigger
  - addAutomationAction
  - updateAutomationAction
  - getAutomations
  - getAutomation
  - putAutomation
  - deleteAutomation
generated: '2026-08-12'
method: generated
source: openapi/_original/bonjoro-api-v2-openapi-original.json
---

# Automate Bonjoro greet creation from a connected service

An **automation** in Bonjoro is a `trigger` + `action` pair bound to a connected service account (a
CRM or email platform). It is what turns "someone signed up in HubSpot" into "a Bonjoro task appears in
the queue". All operationIds below exist verbatim in `openapi/bonjoro-api-v2-openapi.yml`.

Authenticate first (`authenticate`, then `Authorization: Bearer <token>`).

## 1. Discover what can be connected — `getAutomationServices`

`GET /api/v2/automation/services` lists the integrable services and their
`automationServiceDefaultTrigger`. `GET /api/v2/automation/services/{service_id}`
(`getAutomationService`) returns one. Two helpers exist for contact mapping:

- `GET /api/v2/automation/services/{service_id}/contacts` — `getAutomationSericeContacts`
- `POST /api/v2/automation/services/{service_id}/contacts/parameters` —
  `getAutomationSericeContactParameters`

(Both operationIds are misspelled upstream — `Serice`. Use them exactly as published.)

## 2. Check the connected accounts — `getAutomationAccounts`

`GET /api/v2/automation/accounts`, `GET /api/v2/automation/accounts/{account_id}`, and
`DELETE /api/v2/automation/accounts/{account_id}`. Before relying on an account, call
`GET /api/v2/automation/accounts/{account_id}/test` (`testAutomationAccount`) — it validates the stored
credential against the remote service.

## 3. Create the trigger — `postAutomationTrigger`

`POST /api/v2/automation/triggers` creates a trigger; `PUT /api/v2/automation/triggers/{trigger_id}`
(`putAutomationTrigger`) updates it; `GET /api/v2/automation/triggers` (`getAutomationTriggers`) lists
them. The `automationTrigger` schema binds to a connected service through `account_id`.

## 4. Attach the action — `addAutomationAction`

`POST /api/v2/automation/actions` and `PUT /api/v2/automation/actions/{action_id}`
(`updateAutomationAction`). The action is what creates the greet.

## 5. Manage the assembled rule — `getAutomations`

`GET /api/v2/automations` returns `automations`; each `automation` carries its `trigger` and `action`.
`GET/PUT/DELETE /api/v2/automations/{automation_id}` read, update and remove one.

A parallel **editor** surface exists for building these in a UI — `getEditorAutomation`,
`getAutomationForEditor`, `updateAutomationEditor`, `deleteAutomationEditor`, `getEditorActions`,
`getEditorAction`, `getEditorParameter`, `getEditorServices`. Use the non-editor operations for
programmatic control; the editor ones return UI-shaped parameter descriptors.

> Contract caution: `getEditorServices` is published as the operationId for **two** different paths
> (`/api/v2/automation/editor/services` and `/api/v2/automation/editor/triggers`). A generated client
> will collapse them. Call the paths directly.

## 6. Verify it fired

New greets land in the task queue — read them with `getGreetTodos`
(`GET /api/v2/greets/todo`) or `getGreets`. Automation-created greets appear in the same
`event`/`getEvents` stream as manual ones.

## Safety notes for agents

- `deleteAutomation`, `deleteAutomationEditor` and `deleteAutomationAccount` are destructive and have
  **no** confirmation semantics, no soft delete, and no idempotency key. Read before you delete.
- `bulkCreateGreets` accepts up to 200 recipients per greet and has no idempotency guard — a retry
  after a timeout sends duplicate video tasks to real customers. Reconcile with `getGreets` instead of
  retrying.
