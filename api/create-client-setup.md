---
name: create-client-setup
type: setup
version: 1.1.0
collection: client-intelligence
description: Setup for the create-client task — no member-specific parameters. Validates that the collection is installed and dependent read tasks are reachable.
target: create-client
target_type: task
upgrade_compatible: true
---

## Setup Overview

Create Client requires no member-specific configuration. The task collects all inputs at invocation time. This setup verifies the collection is installed, confirms the `list-templates` and `view-template` task dependencies are reachable, registers the alias `@ai:create-client`, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before clients can be created."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*
- Dependent tasks are installed: `list-templates`, `view-template`. If missing from `member-index.json`, instruct: *"Create Client depends on List Templates and View Template. Reinstall the collection so all V1 tasks are present."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Confirm `list-templates` and `view-template` are installed.
4. Register entry in `member-index.json` with alias `@ai:create-client`.
5. Confirm to member: *"Create Client is ready. Say `@ai:create-client` to add a new client. You'll pick a template, fill out fields, name the client, and (optionally) grant other members access at creation time. Before any access change I'll show you exactly what will change and ask you to confirm."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → 1.1.0: access changes now go through the core permission-changes procedure (agent-index-core 3.32.0+); the helper-binary pre-check is gone. No member action needed.
