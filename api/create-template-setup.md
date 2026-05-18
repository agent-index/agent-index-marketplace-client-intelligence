---
name: create-template-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the create-template task — minimal member-level setup. All configuration is inherited from the collection setup; this setup only registers the alias and validates that the collection is installed.
target: create-template
target_type: task
upgrade_compatible: true
---

## Setup Overview

Create Template requires no member-specific configuration. The task interviews the admin for template details at invocation time; nothing needs to be set up ahead of time on the member's side. This setup verifies the collection is installed, registers the alias `@ai:create-template`, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before templates can be authored."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*

---

## Parameters

No member-defined parameters. All configuration is inherited from the collection setup. The task itself collects template-specific inputs (slug, display name, fields) interactively at invocation time.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Register entry in `member-index.json` with alias `@ai:create-template`.
4. Confirm to member: *"Create Template is ready. Say `@ai:create-template` to author a new client template. Note: only members with admin access on the client-intelligence collection can complete the operation — the filesystem will reject the write if you don't have it."*

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
