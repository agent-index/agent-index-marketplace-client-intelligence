---
name: view-permissions
type: task
version: 2.0.0
collection: client-intelligence
description: Show who can access a client and how. Org-public clients report uniform org access (by design — no per-instance ACL exists). Private clients report the pointer's declared scope cross-checked against live Drive permissions (aifs_get_permissions on the owner's folder — readable by the owner and grantees), flagging any drift between the two.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/public-index/, id:{folder_id}/"
writes_to: null
---

## About This Task

View Permissions answers "who can see/edit this client?" per tier:

- **Org-public:** the answer is structural — *every org member, uniformly (writer), via the install-time commons grant*. Render that, plus the owner and the changelog-attribution note. No ACL read needed; optionally confirm the commons grant exists (`aifs_get_permissions` on `instances/`) when the caller doubts provisioning.
- **Private:** two sources, cross-checked:
  1. The pointer's declared `scope` (readers / collaborators) — what agent-index believes.
  2. Live `aifs_get_permissions("id:{location.folder_id}")` — what Drive enforces. Works for the owner and grantees; a caller with no access gets the floor view instead ("private to {owner}").
  Any mismatch (a grant Drive has that the pointer doesn't, or vice versa) is rendered as **drift** with a reconciliation suggestion (owner re-runs grant/revoke-permission, which repairs the pointer through the hard gate).

Permission HISTORY remains out of scope (post-V1, `aifs_get_audit`); current state only. Grant changes are owner-run (`grant-permission` / `revoke-permission`) — this task never modifies anything.

## Workflow

### Step 1: Pre-flight

Auth + installed checks.

### Step 2: Resolve via pointer

Slug or name. Missing → list-clients guidance. Stub → follow the pointer.

### Step 3: Tier-resolved report

Org-public → structural answer (above). Private → declared scope + live read; for each subject render email, Drive role, source (the 2.0 model grants explicitly on the owned folder — inherited entries beyond the owner's ownership are themselves drift worth flagging), permission ID. Cross-check table:

```
Subject                Pointer says     Drive says    Status
jeff@agent-index.ai    reader           reader        OK
jrohwer@gmail.com      collaborator     writer        OK
sam@agent-index.ai     —                reader        DRIFT (granted outside agent-index)
```

`owner_departed: true` → banner: access is live but no longer governable through agent-index.

### Step 4: Suggestions

On drift: *"{owner} can reconcile: `@ai:grant-permission` / `@ai:revoke-permission` repair the pointer as they apply verified changes."* On no access (caller can't read the folder): floor view + ask-the-owner.

## Directives

### Constraints

- Read-only; never modifies grants or pointers (even on drift — reconciliation is the owner's verified action, not a silent fix).
- Never bypass the floor.

### Edge Cases

- `scope: "revoked"`/fully-private with active status → "only {owner}"; verify against the live read if caller is the owner.
- get_permissions fails for a granted reader (rare propagation lag) → render declared scope with a "live check unavailable" note.
- Org-tier caller asks "but can I gate it?" → explain the tier deal, point at transition-client.
