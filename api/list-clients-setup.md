---
name: list-clients-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the list-clients task — no member-specific parameters. Validates that the collection is installed.
target: list-clients
target_type: task
upgrade_compatible: true
---

## Setup Overview

List Clients requires no member-specific configuration. The task enumerates `/shared/client-intelligence/public-index/instances/` (universally readable) and probes per-instance visibility. This setup verifies the collection is installed, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before clients can be listed."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:list-clients`.
4. Confirm to member: *"List Clients is ready. Say `@ai:list-clients` to see every client in the collection. Clients you have view permission on show their full row; others show name-only — that's the visibility floor."*

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
