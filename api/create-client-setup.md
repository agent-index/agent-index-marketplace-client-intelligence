---
name: create-client-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the create-client task — no member-specific parameters. Validates that the collection is installed, the permission-change-helper is available, and dependent read tasks are reachable.
target: create-client
target_type: task
upgrade_compatible: true
---

## Setup Overview

Create Client requires no member-specific configuration. The task collects all inputs at invocation time. This setup verifies the collection is installed, confirms the permission-change-helper binary is reachable, confirms the `list-templates` and `view-template` task dependencies are reachable, registers the alias `@ai:create-client`, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before clients can be created."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*
- Permission-change-helper binary is installed: check for `mcp-servers/permission-helper-go/agent-index-show-plan` or `mcp-servers/permission-helper/show-plan.sh`. If neither: *"The permission-change-helper isn't installed. Run `@ai:update` to complete the core install."*
- Dependent tasks are installed: `list-templates`, `view-template`. If missing from `member-index.json`, instruct: *"Create Client depends on List Templates and View Template. Reinstall the collection so all V1 tasks are present."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Confirm the permission-change-helper binary is present.
4. Confirm `list-templates` and `view-template` are installed.
5. Register entry in `member-index.json` with alias `@ai:create-client`.
6. Confirm to member: *"Create Client is ready. Say `@ai:create-client` to add a new client. You'll pick a template, fill out fields, name the client, and (optionally) grant other members access at creation time. Permission changes go through your browser for explicit Accept."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: migration notes will be added here as new versions of the task are published. Notable: when access-control Phase 5 ships the `inherit` extension to the helper spec, this task's permission application logic will gain the ability to truly narrow per-instance ACLs. Existing instances created in V1 will continue to operate; new instances created post-Phase-5 will benefit from real data visibility floor.
