---
name: edit-default-permissions-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the edit-default-permissions task — no member-specific parameters. Validates that the collection is installed.
target: edit-default-permissions
target_type: task
upgrade_compatible: true
---

## Setup Overview

Edit Default Permissions requires no member-specific configuration. The task interviews the admin for the new policy shape at invocation time and writes the result. This setup verifies the collection is installed, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before the default policy can be edited."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:edit-default-permissions`.
4. Confirm to member: *"Edit Default Permissions is ready. Say `@ai:edit-default-permissions` to change the default policy applied to new clients. Note: only admins can complete the operation."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: any change to the policy schema (new target types, new permission models) will be accompanied by a migration script in `/upgrade/` that rewrites existing policies into the new shape. The `version` field inside `default-permissions.json` is the schema-version anchor.
