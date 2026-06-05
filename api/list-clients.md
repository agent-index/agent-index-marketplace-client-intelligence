---
name: list-clients
type: task
version: 2.0.0
collection: client-intelligence
description: Member-facing read task that renders the org-wide client directory from the universal-floor pointer index. Every client's name, owner, tier, and status are visible to all members; data access is shown per the pointer's scope — mine / shared-with-me / org-public / name-only. Pointer-driven; no per-instance probing required.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/public-index/"
writes_to: null
---

## About This Task

List Clients is the org-wide client directory — the universal visibility floor rendered as a table. It enumerates `public-index/instances/` (the pointer index) and derives everything from the pointers themselves: who owns each relationship, which tier it lives in, and what the caller can do with it. No per-instance stat probing (the 1.x approach) — the pointer's `scope` already answers visibility:

- `org_public` → caller can view and edit (uniform commons access)
- caller is `owner_hash` → **mine**
- caller's email in `scope.readers` / `scope.collaborators` → **shared with me** (read / read+write)
- otherwise → **name-only** (the floor: name, owner, template, status — data gated)

### Inputs

- **`filter_status`** (optional) — `active` (default) / `archived` / `all`. Archived includes `archived-moved-private` commons stubs.
- **`filter_tier`** (optional) — `mine` / `shared-with-me` / `org-public` / `all` (default).
- **`filter_template`**, **`filter_name_prefix`**, **`sort_by`** — as 1.x.

### Outputs

A table: Slug · Name · Owner · Tier/Access · Status · Template · Created. Departed-owner rows annotated. Ends with next-action hints.

## Workflow

### Step 1: Pre-flight

`aifs_auth_status` (re-auth/halt); collection installed check. Read local `member-index.json` for `member_hash` + email.

### Step 2: Enumerate and read pointers

`aifs_list("/shared/client-intelligence/public-index/instances/")` → read each `{slug}.json`. Skip-and-note unparseable entries; never halt on one bad pointer. Permission-denied on the listing → install/provisioning broken; name the admin fix (`@ai:install-collection client-intelligence`).

### Step 3: Derive access per row

Tag each row from the pointer alone (rules above). `owner_departed: true` → annotate "(owner departed — access governed outside agent-index{; adoptable by current recipients}" for private-tier rows. `scope: "revoked"` with active status → treat as name-only.

### Step 4: Filters and sort

As 1.x, plus `filter_tier`. Default hides `archived`/`archived-moved-private`.

### Step 5: Render

```
| Slug | Name | Owner | Access | Status | Template | Created |
|---|---|---|---|---|---|---|
| acme-pharma | Acme Pharma | Bill | org-public (edit) | active | pharma-client v3 | 2026-05-13 |
| globex | Globex | testproduction | shared with me (read) | active | pharma-client v3 | 2026-06-01 |
| falcon | Project Falcon | jeff | name-only | active | pharma-client v3 | 2026-06-02 |
```

Next-action hints: view (`@ai:view-client {slug}`), create, and for name-only rows: *"ask {owner} to share it (`@ai:grant-permission` is owner-run)"*.

### Optional integrity pass (on request or anomaly)

If the member asks for a health check (or a view-client just failed against a listed row): for **mine** rows, `aifs_stat("id:{location.folder_id}")`; for org-public rows, `aifs_exists(location.path + "instance.json")`. Report missing-data rows in a "data-integrity warnings" section. Never probe other members' private folders (it would fail by design, telling us nothing).

## Directives

### Behavior

Read-only; the floor is the headline UX — name-only rows are the design, not an error. Broad questions → filters; narrow questions about one client → chain into view-client.

### Constraints

- **Never write; never call permission ops; never bypass the floor** (no side-channel reads on name-only rows).
- **Never enumerate `instances/` or member spaces directly** — the pointer index is the directory. (Commons enumeration would miss every private client; member spaces can't be enumerated at all.)

### Edge Cases

- Empty index → "No clients yet. Create one with `@ai:create-client`."
- Pointer references a deleted template → render with recorded slug; flag in notes.
- All rows name-only → note: *"You have data access on 0 of {N} clients — they're all private to their owners."*
- Transient read errors → partial table + note.
