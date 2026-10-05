---
name: add-admin-setup
type: setup
version: 1.1.0
collection: client-intelligence
description: Setup for the add-admin task — no member-specific parameters. Validates that the collection is installed.
target: add-admin
target_type: task
upgrade_compatible: true
---

## Setup Overview

Add Admin requires no member-specific configuration. The task collects the target member at invocation time and routes the share through the core permission-changes procedure. This setup verifies the collection is installed, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before admins can be added."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:add-admin`.
4. Confirm to member: *"Add Admin is ready. Say `@ai:add-admin {member}` to grant admin role on client-intelligence. Note: only existing admins can complete the operation; before any change I'll show you exactly what will change and ask you to confirm."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → 1.1.0: access changes now go through the core permission-changes procedure (agent-index-core 3.32.0+; `confirmer: "collection_admin"` — existing collection admins, enforced by the backend, no org-admin check); the helper-binary pre-check is gone. No member action needed.
