---
name: revoke-permission
type: task
version: 2.0.0
collection: client-intelligence
description: Revoke a member's access on a PRIVATE-tier client — symmetric inverse of grant-permission. Owner-only; the unshare goes through permission-change-helper with the owner's Accept, and the pointer scope is updated only after verification. Org-public clients have uniform access; revocation there means transitioning the client private first.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills:
    - permission-change-helper
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
  - permission-change-helper binary 0.4.1+
reads_from: "/shared/client-intelligence/public-index/, id:{member_folder_id}/clients/"
writes_to: "/shared/client-intelligence/public-index/"
---

## About This Task

Revoke Permission removes a member's access on a **private-tier** client. Owner-only, via permission-change-helper (`op: "unshare"`) with the owner's Accept — the agent never calls `aifs_unshare` directly. The pointer's `scope` is updated only after the revocation is verified.

The 1.x "creator cannot be revoked" guardrail is now structural: the owner's access is Drive **ownership** of a folder in their own My Drive — there is no grant to revoke. Revoking "the owner" is meaningless and refused.

On an **org-public** client: uniform access means there is nothing to revoke per-person — the task explains and offers `@ai:transition-client {slug}` (go private, then grant selectively).

### Inputs

- **`slug`** (required) — resolved via the universal-floor pointer.
- **`member`** (required) — who to revoke; resolved against `members-registry.json`.

## Workflow

### Step 1: Pre-flight

As grant-permission Step 1 (auth, installed, helper 0.4.1+).

### Step 2: Resolve client, tier, authority

Read the pointer. `org_public` → explain uniform access, offer transition-client, halt. Private → **owner-only** (same refusal text pattern as grant-permission). Archived → surface, halt.

### Step 3: Resolve target

Registry lookup. Target == owner → refuse: *"You own this client — there's no grant to revoke. To remove it entirely, see `@ai:delete-client`."* Target not in the pointer's `scope` → no-op notice: *"`{target}` doesn't have access to `{name}`."* Exit.

### Step 4: Apply the revocation (owner Accepts)

ONE spec: `op: "unshare"`, resource **`id:{location.folder_id}`** (bare ID), recipient = target email. Owner Accepts.

**HARD GATE:** outcome `"applied"`, OR independent `aifs_get_permissions("id:{folder_id}")` confirming the target is gone (missing/lifecycle-valued outcome file despite confirmed Accept).

### Step 5: Update the pointer + changelog

Remove the target from `readers`/`collaborators`. If both lists are now empty, set `scope: "private"`. Refresh `last_updated`. Append `permission_revoked` to `id:{folder_id}/changelog.json`.

### Step 6: Confirm

*"`{target}` no longer has access to `{name}`.{ The client is now fully private.}"*

## Directives

### Constraints

- **Owner-only; never `aifs_unshare` directly; pointer never claims a state Drive doesn't back.**
- Never delete the pointer or any /shared file (soft semantics).
- Revoking everyone ≠ archiving: the client stays `active` and fully private. Archival is `@ai:delete-client`.

### Edge Cases

- Drive shows a grant the pointer doesn't (drift from out-of-band sharing): Step 4's verification read surfaces it; offer to revoke that too and reconcile the pointer.
- Outcome `rejected` → nothing changed; say so.
- Target's grant exists on Drive but unshare returns RECIPIENT_NOT_FOUND-class errors (account deleted): relay verbatim; pointer may be reconciled to match Drive on the owner's confirmation.
