---
name: grant-permission-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the grant-permission task — no member-specific parameters. Validates that the collection is installed and the permission-change-helper binary is available.
target: grant-permission
target_type: task
upgrade_compatible: true
---

## Setup Overview

Grant Permission requires no member-specific configuration. The task collects target member and permissions at invocation time and routes the actual share through the permission-change-helper. This setup verifies the collection is installed, confirms the helper binary is reachable, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before permissions can be granted."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*
- Permission-change-helper binary is installed (Go preferred, Node fallback). If neither: *"The permission-change-helper isn't installed. Run `@ai:update` to complete the core install."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Confirm the permission-change-helper binary is present.
4. Register entry in `member-index.json` with alias `@ai:grant-permission`.
5. Confirm to member: *"Grant Permission is ready. Say `@ai:grant-permission {slug} {member}` to share access on a client. The actual share goes through your browser for explicit Accept."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: when the helper spec gains `inherit` pass-through, grants will start truly narrowing instance ACLs (rather than being additive on top of the all-members Writer). Existing grants will continue to work; new grants benefit automatically.
