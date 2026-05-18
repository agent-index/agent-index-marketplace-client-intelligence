---
name: revoke-permission-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the revoke-permission task — no member-specific parameters. Validates that the collection is installed and the permission-change-helper binary is available.
target: revoke-permission
target_type: task
upgrade_compatible: true
---

## Setup Overview

Revoke Permission requires no member-specific configuration. This setup verifies the collection is installed, confirms the helper binary is reachable, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before permissions can be revoked."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*
- Permission-change-helper binary is installed. If not: *"The permission-change-helper isn't installed. Run `@ai:update` to complete the core install."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Confirm the permission-change-helper binary is present.
4. Register entry in `member-index.json` with alias `@ai:revoke-permission`.
5. Confirm to member: *"Revoke Permission is ready. Say `@ai:revoke-permission {slug} {member}` to remove access. The actual unshare goes through your browser for explicit Accept. Note: the client's creator cannot be revoked."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: when the helper spec gains `inherit` pass-through, revokes will start meaningfully removing access (rather than no-ops against inherited grants). Existing revokes continue to work as before; new revokes benefit automatically.
