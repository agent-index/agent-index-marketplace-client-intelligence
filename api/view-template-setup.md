---
name: view-template-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the view-template task — no member-specific parameters. Validates that the collection is installed and the member can read the templates folder.
target: view-template
target_type: task
upgrade_compatible: true
---

## Setup Overview

View Template requires no member-specific configuration. It reads a named template plus its version history from `/shared/client-intelligence/templates/`. This setup verifies the collection is installed, registers the alias `@ai:view-template`, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before templates can be viewed."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:view-template`.
4. Confirm to member: *"View Template is ready. Say `@ai:view-template {slug}` to see a specific template's fields and version history."*

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
