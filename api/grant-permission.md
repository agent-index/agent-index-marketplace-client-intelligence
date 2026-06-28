---
name: grant-permission
type: task
version: 2.3.0
collection: client-intelligence
description: Grant access on a PRIVATE-tier client to a member — share (reader) or collaborator (read+write). Owner-only — the grant is a Drive permission on a folder the owner owns in their own My Drive, applied through permission-change-helper with the owner's Accept. Org-public clients have uniform access by design; this task explains and offers transition-client instead.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills:
    - permission-change-helper
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
  - permission-change-helper binary 0.4.1+ (bare id:{folderId} resources)
reads_from: "/shared/client-intelligence/public-index/, id:{member_folder_id}/clients/"
writes_to: "/shared/client-intelligence/public-index/"
---

## About This Task

Grant Permission adds a member's access to a **private-tier** client. Two levels, matching the org-wide sharing vocabulary: *share with X* = X can read; *make X a collaborator* = X can read and write. The grant is a Drive permission on the client folder in the **owner's own My Drive** — only the owner can apply it (they own the folder), via permission-change-helper with their Accept. The pointer's `scope` is updated only after the grant is verified.

The 1.x View/Edit/Delete model is retired with the per-instance-ACL machinery: there is no separate "delete" grant (deletion semantics are soft and owner/admin-bound), and authority is no longer "any current grantee can re-grant" — **the owner is the only grantor** (Drive ownership is the authority; recipients of a reader/writer grant on a My Drive folder cannot re-share it unless Drive's writersCanShare applies, and the task-level rule is owner-only regardless).

On an **org-public** client this task does not apply: access is uniform for all members. It explains that and offers `@ai:transition-client {slug}` if the caller (owner) wants the client moved to the private tier first.

### Inputs

- **`slug`** (required) — the client. Resolved via the universal-floor pointer.
- **`member`** (required) — email or display name; resolved against `members-registry.json`.
- **`level`** (required) — `read` (Drive `reader`) or `collaborate` (Drive `writer`).

### Outputs

- One Drive grant on `id:{client_folder_id}` (owner Accepts).
- Pointer `scope` updated at `/shared/client-intelligence/public-index/instances/{slug}.json`.

## Workflow

### Step 1: Pre-flight

`aifs_auth_status` (re-auth/halt); collection installed check; helper binary present (0.4.1+ — on a `validation_error` mentioning `id:` resources later, the binary is outdated → `@ai:update`).

### Step 2: Resolve the client and tier

Read the pointer at `/shared/client-intelligence/public-index/instances/{slug}.json` (resolve by name via the index if the caller gave a name). Halt if missing.

- `scope == "org_public"` → *"`{name}` is org-public — every member already has access (uniform, by design; per-person gating doesn't exist in the org tier). If you want it gated, first take it private: `@ai:transition-client {slug}`."* Halt cleanly.
- `status == "archived"` or `scope == "revoked"` with archived status → surface status; offer view-client. Halt.
- Private tier: confirm the **caller is the owner** (`owner_hash == caller.member_hash`). Non-owner → *"Only the owner ({owner}) can grant access — the client lives in their private space. Ask them, or view what you can: `@ai:view-client {slug}`."* Halt.

### Step 3: Resolve target member

As 1.x (registry lookup, disambiguate, reject unknown). Self-grant → refuse (owner already has everything). Already granted at the requested level (per pointer scope) → no-op notice, exit.

### Step 4: Collect level

`read` or `collaborate`. If the target is currently a reader and the request is collaborate (or vice versa), frame it as a level change — the helper spec is the same single `share` op (Drive updates the role).

### Step 5: Apply the grant (owner Accepts)

Compose ONE permission-change-helper spec: `op: "share"`, resource **`id:{location.folder_id}`** (bare folder ID from the pointer), recipient = target email, role = `reader`|`writer`. Owner reviews and Accepts. Never `aifs_share` directly.

**HARD GATE:** proceed only on outcome `"applied"`, OR — if the outcome file is missing/page-lifecycle-valued despite a confirmed Accept — an independent `aifs_get_permissions("id:{folder_id}")` confirming the grant (helper ≤0.4.0 race; fixed 0.4.1; fallback retained).

### Step 6: Update the pointer

Overwrite the pointer's `scope`: move/add the target in `readers`/`collaborators` (object form `{"readers": [...], "collaborators": [...]}` replaces bare `"private"` on first grant). `last_updated` refreshed. **If the pointer's `location` lacks `item_drive_id`** (a pre-C.1.3 client), `aifs_stat("id:{location.folder_id}")` once and write the returned `drive_id` into `location.item_drive_id` — without it the grantee on OneDrive can discover the client but not open it (C.1.3 `crossdriveread`: the recipient opens via `id:{item_drive_id}:{folder_id}/...`). Append a `permission_granted` event to the client's `changelog.json` (`id:{folder_id}/changelog.json`).

### Step 7: Confirm

*"`{target}` can now {read|read and write} `{name}`. They'll find it via `@ai:list-clients` (shared-with-me)."*

## Directives

### Constraints

- **Owner-only.** No re-granting by recipients, no admin substitution (F12 lesson: validate and operate as the role the model assigns).
- **Never `aifs_share` directly; never per-instance ACLs in the org tier.**
- **Pointer scope must never claim a grant the Drive state doesn't back** (hard gate).
- Soft semantics on the pointer (overwrite-only; it lives on the Shared Drive).

### Edge Cases

- Pointer exists but `aifs_stat("id:{folder_id}")` fails for the owner → folder moved/deleted outside agent-index; surface and suggest `@ai:list-clients` reconcile.
- `partial_failure` (multi-grant future use) → pointer reflects only verified grants.
- Owner's helper outcome `rejected` → nothing granted, nothing written; say so.
- Target not yet a Drive account (consumer email typo etc.) → helper surfaces INVALID_RECIPIENT; relay verbatim.
