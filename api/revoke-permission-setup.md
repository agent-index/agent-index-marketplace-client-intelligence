---
name: revoke-permission-setup
type: setup
version: 1.1.0
collection: client-intelligence
description: Setup for the revoke-permission task — no member-specific parameters. Validates that the collection is installed.
target: revoke-permission
target_type: task
upgrade_compatible: true
---

## Setup Overview

Revoke Permission requires no member-specific configuration. This setup verifies the collection is installed, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before permissions can be revoked."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:revoke-permission`.
4. Confirm to member: *"Revoke Permission is ready. Say `@ai:revoke-permission {slug} {member}` to remove access. Before any change I'll show you exactly what will change and ask you to confirm. Note: the client's creator cannot be revoked."*

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
