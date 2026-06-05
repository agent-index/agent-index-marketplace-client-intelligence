---
name: edit-default-permissions
type: task
version: 2.0.0
collection: client-intelligence
description: Admin task — repurposed in 2.0. Sets the org's default-visibility behavior for client creation (private_first / ask / org_public_first), written as an org-mandated setup value. No ACL policy exists anymore — the 1.x default-permissions.json grants machinery is retired with the per-instance ACL model.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/"
writes_to: "/shared/client-intelligence/setup/"
---

## About This Task

Repurposed in 2.0.0: there is no default ACL policy to edit anymore. Creation-time access is structural — private = owner-only until granted; org-public = uniform. What the org CAN set is **how create-client asks the visibility question**:

- `private_first` (default) — private is the default answer; org-public must be chosen explicitly.
- `ask` — neutral prompt, no default.
- `org_public_first` — org-public is the default answer (for orgs that treat the commons as primary and private as the exception).

The value is the org-mandated `clients_default_visibility` in the collection's `setup/collection-setup-responses.md` — the same file the install/upgrade interview writes; members receive it through the Phase 4.5 step-9 reconcile on their next `@ai:update`.

Admin-only (org-config `admins[]` check — same pre-flight as other admin tasks). The 1.x `config/default-permissions.json` is retired; if present, this task offers to mark it `{"retired": "2.0.0"}`.

## Workflow

### Step 1: Pre-flight

Auth + installed checks; caller must be in org-config `admins[]` (refuse otherwise).

### Step 2: Show current value and options

Read `setup/collection-setup-responses.md` → current `clients_default_visibility`. Present the three options with the custody framing (revenue-bearing org relationships → commons; prospecting/sensitive → private).

### Step 3: Confirm and write

Explicit confirmation, then update the org-mandated value in `collection-setup-responses.md` (revision-aware write; preserve everything else in the file). If the legacy `config/default-permissions.json` exists, offer the retired-marker overwrite.

### Step 4: Confirm

*"Default visibility prompt is now `{value}`. Members pick it up on their next `@ai:update` (setup-responses reconcile); create-client uses it immediately for anyone already current."*

## Directives

### Constraints

- Admin-only. Never writes Drive ACLs (nothing here to write). Never touches existing clients or pointers — this shapes the PROMPT for future creations only.
- Preserve all other setup-responses content on write.

### Edge Cases

- Setup-responses missing (broken install) → halt; `@ai:install-collection client-intelligence` guidance.
- Legacy default-permissions.json present with custom grants an admin cares about: surface its content before retiring it; the equivalent intent in 2.0 is per-creation grants (private tier) or the org tier itself.
