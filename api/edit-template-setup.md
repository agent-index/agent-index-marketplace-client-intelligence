---
name: edit-template-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the edit-template task — no member-specific parameters. Validates that the collection is installed and that view-template is available as a dependency.
target: edit-template
target_type: task
upgrade_compatible: true
---

## Setup Overview

Edit Template requires no member-specific configuration. The task interviews the admin for the edits at invocation time and computes the migration impact from the live filesystem state. This setup verifies the collection is installed, confirms the `view-template` task dependency is present, registers the alias `@ai:edit-template`, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before templates can be edited."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*
- The `view-template` task is installed and reachable. Check via `member-index.json`. If missing, instruct: *"Edit Template depends on View Template for the live-state read. Install or repair the collection so all V1 tasks are present."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Confirm `view-template` is installed.
4. Register entry in `member-index.json` with alias `@ai:edit-template`.
5. Confirm to member: *"Edit Template is ready. Say `@ai:edit-template {slug}` to edit a client template. Note: only members with admin access on client-intelligence can complete the operation. If you choose Migrate, existing instances will be updated to the new schema — confirm carefully before proceeding."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: migration notes will be added here as new versions of the task are published. Notable future-version concerns: if a future version changes the migration semantics (e.g., introduces lazy migration), the upgrade flow will require re-confirmation on any in-flight edits.
