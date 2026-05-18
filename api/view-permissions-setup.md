---
name: view-permissions-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the view-permissions task — no member-specific parameters. Validates that the collection is installed.
target: view-permissions
target_type: task
upgrade_compatible: true
---

## Setup Overview

View Permissions requires no member-specific configuration. The task calls `aifs_get_permissions` directly (read-only, agent-callable). This setup verifies the collection is installed, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before permissions can be viewed."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:view-permissions`.
4. Confirm to member: *"View Permissions is ready. Say `@ai:view-permissions {slug}` to see who has access to a specific client."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: when `aifs_get_audit` ships in access-control, the related `view-client-audit` task will surface permission history alongside this current-state view. Migration is additive — current task is unchanged.
