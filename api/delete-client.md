---
name: delete-client
type: task
version: 2.0.0
collection: client-intelligence
description: Archive (default) or remove a client. Soft semantics everywhere members can't delete — org-public archival marks status; the floor entry always survives as the historical record. True deletion exists only where someone actually holds delete power — owners in their own My Drive, admins in the commons. Includes unarchive.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/, id:{folder_id}/"
writes_to: "/shared/client-intelligence/, id:{folder_id}/"
---

## About This Task

Delete Client retires a client record. The 2.0 semantics follow the soft-delete conventions (standards.md): **archival is the default and usually the only option**; the universal-floor pointer is NEVER deleted by this task — it carries `status: "archived"` as the historical record ("we once had this relationship") and keeps `list-clients` honest.

What's actually possible, per tier and role:

| Tier | Caller | Archive | Unarchive | True delete |
|---|---|---|---|---|
| org-public | any member | yes (status mark) | yes (status mark, attributed) | no — members can't trash /shared; admins may purge on request (outside this task) |
| private | owner | yes | yes | yes — it's their My Drive ("hard delete": folder removed; pointer goes `status: "deleted"`, name retained) |
| private | collaborator/reader | no — owner-only | no | no |

The 1.x "hard delete" that removed the public-index entry is GONE: the floor record persists by design. An admin who must truly expunge a name (legal erasure) does it manually and documents it.

### Inputs

- `slug` (or name) — resolved via the pointer.
- `mode` — `archive` (default) / `unarchive` / `hard` (private-tier owner only).
- Confirmation: normal for archive/unarchive; for `hard`, an explicit phrase including "permanently".

## Workflow

### Step 1: Pre-flight

Auth + installed checks.

### Step 2: Resolve via pointer; authority check

Org-public: any member may archive/unarchive (attributed). Private: **owner-only** for all modes (collaborators edit data, not lifecycle). Stub (`archived-moved-private`): nothing to do here — point at the live copy.

### Step 3: Confirm

Archive: *"Archive `{name}`? Data stays where it is; it disappears from default listings; anyone with access can still read it. Unarchive anytime."*
Hard (private owner only): *"Permanently delete `{name}` from your Drive? The data is gone for good. The org directory will keep a name-only record marked deleted. Type a confirmation including 'permanently'."*

### Step 4: Apply

**Archive:** append `archived` changelog event (actor-attributed) to the instance changelog (tier-resolved base path); overwrite the pointer `status: "archived"`, `last_updated`. Instance data untouched.

**Unarchive:** symmetric — `unarchived` event; pointer `status: "active"`.

**Hard (private owner):** append nothing (the log is going with the folder); owner deletes the folder in their own My Drive via `aifs_delete("id:{folder_id}")` (their space — allowed); overwrite the pointer: `status: "deleted"`, `scope: "revoked"`, `location: null`, `last_updated`. Any grants die with the folder.

### Step 5: Confirm to caller

What happened, where the record stands, how to reverse (archive) or that it can't be (hard).

## Directives

### Constraints

- **The pointer is never deleted by this task** — floor records are permanent history (admin-only manual purge for legal erasure, documented).
- **Never `aifs_delete` anything under /shared** (members can't anyway; the task must not try).
- Org-tier archive/unarchive must be changelog-attributed (same invariant as edit-client).
- Hard mode: private tier, owner, explicit "permanently" phrase — all three or refuse.
- No writes before confirmation.

### Edge Cases

- Archive an already-archived client → no-op notice.
- Unarchive a `deleted` pointer → impossible; explain (data is gone), suggest re-create.
- Hard delete with collaborators still granted: warn — *"{N} people currently have access; deletion cuts them off immediately."* Proceed on confirmation.
- `owner_departed` private client: nobody can archive/delete it through agent-index (owner gone, org has no power) — pointer stays as annotated; explain honestly.
- Pointer write succeeds but changelog append failed (archive): retry; surface loudly if it won't stick (attribution invariant).
