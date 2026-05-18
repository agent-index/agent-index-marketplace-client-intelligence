---
name: edit-client-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the edit-client task — no member-specific parameters. Validates that the collection is installed and view-client is available.
target: edit-client
target_type: task
upgrade_compatible: true
---

## Setup Overview

Edit Client requires no member-specific configuration. The task interviews the member for edits at invocation time. This setup verifies the collection is installed, confirms the `view-client` task dependency is reachable, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before clients can be edited."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*
- The `view-client` task is installed. If not, instruct: *"Edit Client depends on View Client. Reinstall the collection so all V1 tasks are present."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Confirm `view-client` is installed.
4. Register entry in `member-index.json` with alias `@ai:edit-client`.
5. Confirm to member: *"Edit Client is ready. Say `@ai:edit-client {slug}` to update a client. You'll only be able to complete the edit if you have Edit permission on the instance."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: migration notes will be added here as new versions of the task are published.
