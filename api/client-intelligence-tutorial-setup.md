---
name: client-intelligence-tutorial-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the client-intelligence-tutorial skill — no member-specific parameters. Validates that the collection is installed and the tutorial is reachable.
target: client-intelligence-tutorial
target_type: skill
upgrade_compatible: true
---

## Setup Overview

The client-intelligence tutorial skill requires no member-specific configuration. The skill reads collection state (`/shared/client-intelligence/collection-state.json` and `config/default-permissions.json`) at invocation time to keep examples accurate. This setup verifies the collection is installed, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before the tutorial is useful."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:client-intelligence-tutorial`.
4. Confirm to member: *"The client-intelligence tutorial is ready. Say `@ai:client-intelligence-tutorial` for a guided tour of the collection, or ask a specific question about how something works."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: as the collection gains new tasks or behavior, the tutorial's topics should be expanded accordingly. The migration notes will list what changed in the tutorial.
