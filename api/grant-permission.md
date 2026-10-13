---
name: grant-permission
type: task
version: 2.4.0
collection: client-intelligence
description: Grant access on a PRIVATE-tier client to a member — share (reader) or collaborator (read+write). Owner-only — the grant is a Drive permission on a folder the owner owns in their own My Drive, applied through the core permission-changes procedure with the owner confirming the exact change. Org-public clients have uniform access by design; this task explains and offers transition-client instead.
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

Grant Permission adds a member's access to a **private-tier** client. Two levels, matching the org-wide sharing vocabulary: *share with X* = X can read; *make X a collaborator* = X can read and write. The grant is a Drive permission on the client folder in the **owner's own My Drive** — only the owner can apply it (they own the folder), through the core permission-changes procedure with the owner confirming. The pointer's `scope` is updated only after the grant is verified.

The 1.x View/Edit/Delete model is retired with the per-instance-ACL machinery: there is no separate "delete" grant (deletion semantics are soft and owner/admin-bound), and authority is no longer "any current grantee can re-grant" — **the owner is the only grantor** (Drive ownership is the authority; recipients of a reader/writer grant on a My Drive folder cannot re-share it unless Drive's writersCanShare applies, and the task-level rule is owner-only regardless).

On an **org-public** client this task does not apply: access is uniform for all members. It explains that and offers `@ai:transition-client {slug}` if the caller (owner) wants the client moved to the private tier first.

### Inputs

- **`slug`** (required) — the client. Resolved via the universal-floor pointer.
- **`member`** (required) — email or display name; resolved against `members-registry.json`, or any email address (people outside the org are allowed and flagged).
- **`level`** (required) — `read` (Drive `reader`) or `collaborate` (Drive `writer`).

### Outputs

- One Drive grant on `id:{client_folder_id}` (owner confirms; verified).
- Pointer `scope` updated at `/shared/client-intelligence/public-index/instances/{slug}.json`.

## Workflow

### Step 1: Pre-flight

`aifs_auth_status` (re-auth/halt); collection installed check.

### Step 2: Resolve the client and tier

Read the pointer at `/shared/client-intelligence/public-index/instances/{slug}.json` (resolve by name via the index if the caller gave a name). Halt if missing.

- `scope == "org_public"` → *"`{name}` is org-public — every member already has access (uniform, by design; per-person gating doesn't exist in the org tier). If you want it gated, first take it private: `@ai:transition-client {slug}`."* Halt cleanly.
- `status == "archived"` or `scope == "revoked"` with archived status → surface status; offer view-client. Halt.
- Private tier: confirm the **caller is the owner** (`owner_hash == caller.member_hash`). Non-owner → *"Only the owner ({owner}) can grant access — the client lives in their private space. Ask them, or view what you can: `@ai:view-client {slug}`."* Halt.

### Step 3: Resolve target member

Registry lookup by name or email; disambiguate → their `sharing_identity`. A name that doesn't resolve → ask for their email address (never guess). An address outside the registry is allowed: the permission-changes table marks it **outside your org**, and it's the owner's call. Tell the owner the person opens it from their storage app's "Shared with me", because `@ai:list-clients` is for org members. Self-grant → refuse (owner already has everything). Don't short-circuit on the pointer's `scope` ("already granted"): the pointer may be stale. Always continue — the procedure compares against live Drive state and reports an existing grant as already in effect (`skipped_noop`), and Step 6 then reconciles the pointer (and backfills `item_drive_id`).

### Step 4: Collect level

`read` or `collaborate`. If the target is currently a reader and the request is collaborate (or vice versa), frame it as a level change — it's the same single `share` change (Drive updates the role).

### Step 5: Apply the grant (owner confirms)

Follow `/agent-index-core/internal/permission-changes.md`:
- `calling_task: "grant-permission"`, `confirmer: "owner"`
- one change: `op: "share"`, resource **`id:{location.folder_id}`** (bare folder ID from the pointer), `resource_label` "Client '{name}'", `is_folder: true`, recipient = target's `sharing_identity` (or the email given), role = `reader`|`writer`, `reason` "You asked to share '{name}' with {target} ({read|read and write})", `source: "member_request"`

The owner sees the exact table and confirms it, and the grant is verified.

**Gate:** continue to Step 6 only if the row is `applied` or `skipped_noop` (including `nothing_to_change` — the grant already exists; Step 6 still runs so the pointer is reconciled and `item_drive_id` backfilled). Anything else → nothing is written; tell the owner what happened (the procedure's summary line).

### Step 6: Update the pointer

Recompute the pointer's `scope` from a **fresh** `aifs_get_permissions({path: "id:{location.folder_id}"})` read — not from the change you intended: take the explicit grants, exclude the owner, and map `writer` → `collaborators`, `reader`/`commenter` → `readers`. Overwrite `scope` with the object form `{"readers": [...], "collaborators": [...]}` (it replaces bare `"private"` on first grant). This also corrects a stale pointer. `last_updated` refreshed. **If the pointer's `location` lacks `item_drive_id`** (a pre-C.1.3 client), `aifs_stat("id:{location.folder_id}")` once and write the returned `drive_id` into `location.item_drive_id` — without it the grantee on OneDrive can discover the client but not open it (C.1.3 `crossdriveread`: the recipient opens via `id:{item_drive_id}:{folder_id}/...`). If the row was `applied` (not `skipped_noop`), append a `permission_granted` event to the client's `changelog.json` (`id:{folder_id}/changelog.json`).

### Step 7: Confirm

- Org member (in the registry): *"`{target}` can now {read|read and write} `{name}`. They'll find it via `@ai:list-clients` (shared-with-me)."*
- Outside-org recipient: *"`{target}` can now {read|read and write} `{name}`. They're outside your org, so they won't see it in `@ai:list-clients` — they open it from their storage app's "Shared with me"."*
- Row was `skipped_noop`: *"`{target}` already had {read|read and write} access to `{name}` — nothing changed in Drive. The client's sharing record is now up to date."*

## Directives

### Constraints

- **Owner-only.** No re-granting by recipients, no admin substitution (F12 lesson: validate and operate as the role the model assigns).
- **Never call `aifs_share` outside the core permission-changes procedure; never per-instance ACLs in the org tier.**
- **Pointer scope must never claim a grant the Drive state doesn't back** (Step 5 gate).
- Soft semantics on the pointer (overwrite-only; it lives on the Shared Drive).

### Edge Cases

- Pointer exists but `aifs_stat("id:{folder_id}")` fails for the owner → folder moved/deleted outside agent-index; surface and suggest `@ai:list-clients` reconcile.
- Owner declines the table → nothing granted, nothing written; say so.
- Pointer `scope` already lists the target at the requested level → not a shortcut; run Steps 4–6 anyway (the procedure reports `skipped_noop` if Drive agrees, and Step 6 reconciles the pointer either way).
- Address the backend rejects (typo, or the Workspace blocks external sharing) → the row is `failed` with the reason; relay it.
