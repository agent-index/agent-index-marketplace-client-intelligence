---
name: view-client-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the view-client task — no member-specific parameters. Validates that the collection is installed.
target: view-client
target_type: task
upgrade_compatible: true
---

## Setup Overview

View Client requires no member-specific configuration. The task reads a named client from `/shared/client-intelligence/`. If the caller has view permission on the instance, the full data is rendered; otherwise the visibility-floor view (name only, from public-index) is rendered. This setup verifies the collection is installed, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before clients can be viewed."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:view-client`.
4. Confirm to member: *"View Client is ready. Say `@ai:view-client {slug}` to see a client's full data and recent changelog. Clients you don't have view permission on will show the visibility-floor view (name only)."*

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
