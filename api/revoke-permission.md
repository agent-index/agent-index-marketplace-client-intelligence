---
name: revoke-permission
type: task
version: 2.1.0
collection: client-intelligence
description: Revoke a member's access on a PRIVATE-tier client — symmetric inverse of grant-permission. Owner-only; the unshare goes through the core permission-changes procedure with the owner confirming, and the pointer scope is updated only after verification. Org-public clients have uniform access; revocation there means transitioning the client private first.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/public-index/, id:{member_folder_id}/clients/"
writes_to: "/shared/client-intelligence/public-index/"
---

## About This Task

Revoke Permission removes a member's access on a **private-tier** client. Owner-only, through the core permission-changes procedure (an `unshare` change the owner confirms) — never `aifs_unshare` outside it. The pointer's `scope` is updated only after the revocation is verified.

The 1.x "creator cannot be revoked" guardrail is now structural: the owner's access is Drive **ownership** of a folder in their own My Drive — there is no grant to revoke. Revoking "the owner" is meaningless and refused.

On an **org-public** client: uniform access means there is nothing to revoke per-person — the task explains and offers `@ai:transition-client {slug}` (go private, then grant selectively).

### Inputs

- **`slug`** (required) — resolved via the universal-floor pointer.
- **`member`** (required) — who to revoke; resolved against `members-registry.json`, or the email address they were granted under (people outside the org can hold grants).

## Workflow

### Step 1: Pre-flight

As grant-permission Step 1 (auth, installed).

### Step 2: Resolve client, tier, authority

Read the pointer. `org_public` → explain uniform access, offer transition-client, halt. Private → **owner-only** (same refusal text pattern as grant-permission). Archived → surface, halt.

### Step 3: Resolve target

Registry lookup → their `sharing_identity`, or the email address as given for someone outside the org. Target == owner → refuse: *"You own this client — there's no grant to revoke. To remove it entirely, see `@ai:delete-client`."* (Don't pre-decide "no grant" here; the procedure compares against live Drive state — see the Step 4 result handling.)

### Step 4: Apply the revocation (owner confirms)

Follow `/agent-index-core/internal/permission-changes.md`:
- `calling_task: "revoke-permission"`, `confirmer: "owner"`
- one change: `op: "unshare"`, resource **`id:{location.folder_id}`** (bare ID), `resource_label` "Client '{name}'", `is_folder: true`, recipient = the target's identity, `reason` "You asked to remove {target}'s access to '{name}'", `source: "member_request"`

The owner confirms the removal individually, and it's verified.

**Gate (act on the row state):**

- **`applied`** → Step 5.
- **`skipped_noop`** (nothing explicit to remove) → read `aifs_get_permissions({path: "id:{location.folder_id}", include_inherited: true})` fresh to tell the cases apart:
  - The target still has access via **inheritance** (or a group) → *"`{target}`'s access to `{name}` is inherited and can't be removed here — it comes from a grant on a parent folder or a group, not from this client's own sharing."*
  - The target has no access at all → *"`{target}` doesn't have access to `{name}`."*

  In both cases run Step 5 to reconcile the pointer to the fresh read (no changelog event), then exit without the Step 6 message.
- **`declined`** (including a cancelled table), **`failed`**, **`verify_mismatch`** → nothing written; relay the procedure's summary line and say so.

### Step 5: Update the pointer + changelog

Re-read `aifs_get_permissions("id:{folder_id}")` and set `readers`/`collaborators` to the explicit grants actually in effect (excluding the owner). If none remain, set `scope: "private"`. Refresh `last_updated`. Only when the row was `applied`, append `permission_revoked` to `id:{folder_id}/changelog.json`.

### Step 6: Confirm

*"`{target}` no longer has access to `{name}`.{ The client is now fully private.}"*

## Directives

### Constraints

- **Owner-only; never `aifs_unshare` outside the core permission-changes procedure; pointer never claims a state Drive doesn't back.**
- Never delete the pointer or any /shared file (soft semantics).
- Revoking everyone ≠ archiving: the client stays `active` and fully private. Archival is `@ai:delete-client`.

### Edge Cases

- Drive shows a grant the pointer doesn't (drift from out-of-band sharing): the Step 5 re-read surfaces it and the pointer is reconciled to Drive. Offer to revoke it too (a new run, `source: "current_state"`).
- Owner declines → nothing changed; say so.
- Target's grant exists on Drive but the removal `failed` with a recipient-not-found reason (account deleted): relay it. The gate holds — this run writes nothing. If the owner then asks for the pointer to be brought in line with Drive, that is a **separate reconciliation**, not part of the revocation: a fresh `aifs_get_permissions` read and an overwrite of the pointer's `scope` with exactly what Drive shows (which, while the grant still exists, still lists the target). Never drop the target from the pointer on the strength of the failed removal.
