---
name: list-clients
type: task
version: 1.0.0
collection: client-intelligence
description: Member-facing read task that enumerates all clients in the collection. For each client, shows the name; for clients the caller has view permission on, additionally shows status, template, and creation metadata. Members without view access on a given client see only its name — that's the visibility floor in action.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
reads_from: "/shared/client-intelligence/"
writes_to: null
---

## About This Task

List Clients is the directory of every client in the collection. Names are universally visible (the visibility floor); data is gated per-instance. The task enumerates `public-index/instances/`, surfaces the name + metadata for every client, and for each one probes whether the caller can read the underlying instance data — instances the caller can't read are surfaced as name-only with a `(no view)` annotation.

Authority is filesystem-enforced. Every collection member has read on `public-index/instances/` via the install-time setup grant to the all-members group. The per-instance probe uses `aifs_stat` (cheap) to determine visibility.

### Inputs

- **`filter_status`** (optional) — `active` (default) / `archived` / `all`.
- **`filter_template`** (optional) — only show clients created from a given template slug.
- **`filter_name_prefix`** (optional) — case-insensitive name prefix.
- **`sort_by`** (optional) — `name` (default) / `created` / `last_updated_in_public_index`.

### Outputs

A formatted table of clients surfaced to the caller. For each row:

- Slug
- Name
- Status
- Template (slug + version)
- Created date
- Visibility indicator: full row vs. name-only (visibility floor)

If no clients exist, surface: *"No clients yet. Create one with `@ai:create-client`."*

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with the not-installed message.

### Step 2: Enumerate public-index

`aifs_list("/shared/client-intelligence/public-index/instances/")`. Filter to entries whose name ends in `.json` (drops `.gitkeep` and any non-data files).

If the list returns permission-denied: *"You don't have read access to `/shared/client-intelligence/public-index/instances/`. This usually means the install is broken or the all-members grant was revoked. Contact your org admin."*

If empty, surface the "no clients" message.

### Step 3: Read each public-index entry

For each `{slug}.json`, `aifs_read` and parse. Collect slug, name, template_slug, template_version, created, created_by, status.

If a single entry fails to read or parse, skip it and note the skip in a "skipped clients" section of the output. Don't halt.

### Step 4: Apply filters

- If `filter_status` is set and not `all`, filter rows to matching status.
- If `filter_template` is set, filter rows where `template_slug` matches (case-sensitive).
- If `filter_name_prefix` is set, filter rows where `name` (lowercased) starts with the prefix (lowercased).

### Step 5: Probe per-instance visibility

For each filtered entry, `aifs_stat("/shared/client-intelligence/instances/{slug}/")`. Tag each row:

- **Success** (stat returns metadata): tag `visible: true`.
- **Permission-denied**: tag `visible: false`.
- **Path-not-found**: data-integrity issue (public-index entry without an instance folder). Tag `visible: missing` and note in a separate "broken" section.

Probes can be batched conceptually but execute serially against the adapter (one `aifs_stat` per slug). For collections with many clients, performance is bounded by N stat calls.

### Step 6: Apply sort

Sort the filtered rows per `sort_by` (default: name, ascending, case-insensitive).

### Step 7: Render the table

```
| Slug | Name | Status | Template | Created | Visibility |
|---|---|---|---|---|---|
| acme-pharma | Acme Pharma | active | pharma-client v3 | 2026-05-13 | full |
| globex-pharma | Globex Pharma | active | pharma-client v3 | 2026-04-22 | (no view) |
| internal-codename-1 | Project Falcon | active | pharma-client v3 | 2026-05-01 | (no view) |
```

For rows tagged `(no view)`, render only the slug, name, status, template, and created — the visibility-floor view. The data isn't accessed.

For rows tagged `visible: missing`, render a separate "data-integrity warnings" section:

> *"{N} client(s) have public-index entries but no instance folder. This indicates a partial deletion or a write that didn't complete. An admin can investigate via the filesystem."*

End with next-action suggestions:

> *"To view a client's full data: `@ai:view-client {slug}`. To create a new client: `@ai:create-client`. To request access on a client you can't view: ask a current grantee or admin to run `@ai:grant-permission {slug} {your_email}`."*

---

## Directives

### Behavior

Read-only. The visibility-floor view is the headline UX — members see the full org-wide directory of client names, but data is gated per-instance. Be clear in the rendering that "(no view)" rows are not an error; they're the design.

When asked broad questions ("which clients are pharma-related?"), apply filters where possible (template_slug, name_prefix) and run the task. When asked narrow questions about one specific client, chain into `@ai:view-client` rather than listing everything.

### State Management

Not stateful.

### Constraints

- **Never write anything.**
- **Never call permission-modifying ops.**
- **Never bypass the visibility floor** — for `visible: false` rows, do not attempt to read data through any side channel. The name-only view is the entire result for those rows.
- **Never enumerate `instances/` directly** — enumerate `public-index/instances/`. The public-index is the authoritative directory for name-level discovery; `instances/` requires per-folder access and would silently omit clients the caller can't read.

### Edge Cases

- **Public-index is empty.** Step 2 catches it. Surface the "no clients yet" message.
- **All clients are `visible: false`.** Surface the name-only table with a note: *"You have view permission on 0 of {N} clients. Use `@ai:grant-permission` requests to ask for access on specific clients you need."*
- **Public-index entry references a template that doesn't exist (deleted or renamed).** Show the row with the recorded `template_slug` even if the template is gone; flag in a note section.
- **A status filter excludes everything.** Surface: *"No clients match the filter `status={status}`. Try `status=all` to see archived clients too."*
- **Filesystem returns transient errors mid-enumeration.** Surface what was successfully listed and a note about the partial result.
