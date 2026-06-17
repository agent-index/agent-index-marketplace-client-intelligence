---
name: list-clients
type: task
version: 2.1.0
collection: client-intelligence
description: Member-facing read task that renders the org-wide client directory from the universal-floor pointer index. Every client's name, owner, tier, and status are visible to all members; data access is shown per the pointer's scope — mine / shared-with-me / org-public / name-only. Pointer-driven; no per-instance probing required. When a brand-book provider supplies the `client-list` template, renders a branded, interactive Cowork directory (cards with Add/Edit/Archive) by default; otherwise renders the markdown table. Can export the directory to a standalone HTML / PDF / Word artifact on request.
stateful: false
produces_artifacts: true
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
optional_capabilities:
  - "brand-book (get-template/get-element/get-brand-guidelines) — enables the branded interactive directory view and standalone exports; degrades to the markdown table when absent (see /internal/resolve-brand.md)"
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

The same row set, rendered on one of two surfaces:

- **Branded interactive directory (default in Cowork)** — when a brand-book provider supplies a `client-list` template: an interactive directory composed from that template's own elements (per-item cards under a header), with the template's action buttons wired to live flows — page-level and per-card actions mapped to this collection's tasks (Step 6a). This is the headline surface; it carries the same access semantics as the table (name-only rows show no data, only the floor).
- **Markdown table (fallback / non-Cowork / on request)** — Slug · Name · Owner · Tier/Access · Status · Template · Created. Departed-owner rows annotated; ends with next-action hints. Always available via "list clients as a table".

On request ("export this list", "save the client directory") the directory is rendered to a **standalone artifact** — HTML page, PDF, or Word — using the brand-book renderings (action buttons omitted in static formats).

## Workflow

### Step 1: Pre-flight

`aifs_auth_status` (re-auth/halt); collection installed check. Read local `member-index.json` for `member_hash` + email.

### Step 2: Enumerate and read pointers

`aifs_list("/shared/client-intelligence/public-index/instances/")` → read each `{slug}.json`. Skip-and-note unparseable entries; never halt on one bad pointer. Permission-denied on the listing → install/provisioning broken; name the admin fix (`@ai:install-collection client-intelligence`).

### Step 3: Derive access per row

Tag each row from the pointer alone (rules above). `owner_departed: true` → annotate "(owner departed — access governed outside agent-index{; adoptable by current recipients}" for private-tier rows. `scope: "revoked"` with active status → treat as name-only.

### Step 4: Filters and sort

As 1.x, plus `filter_tier`. Default hides `archived`/`archived-moved-private`.

### Step 5: Resolve brand (inline — int4)

Resolution is inlined here rather than referencing `/internal/resolve-brand.md`, because members cannot read collection `/internal/` files at runtime (not synced locally; not path-resolvable on the org remote — bug 20260608 int4). `/internal/resolve-brand.md` remains authoring documentation only. The steps:

1. **Find the provider.** `aifs_read("/org-config.json")` → `capability_providers["brand-book"].providers`. Empty/absent → **no brand**: go to Step 6b (table) with a one-line notice ("Brand styling not configured — install and register a brand-book provider to enable it"). Exactly one → use it. Multiple → V1 isn't configured for multi-provider binding; go to Step 6b unbranded with a notice.
2. **Resolve the provider's read base (id-anchored, core 3.10.1).** Find the provider collection in `installed_collections[]`; `base = "id:{folder_id}"` if present, else `/{provider_collection}`. `brand_book_version` = the provider's `capability_version` from the registry (authoritative; do NOT name-resolve the provider's collection.json — bug db13).
3. **Fetch once (read-only provider ops, executed from `{base}/api/...`):** `get-template("client-list", "html")` — `client-list` is THIS collection's directory artifact type; whatever brand template the org published under that name is used (none, or any failure → Step 6b unbranded). For **each element named in the returned template's `sections`** (the consumer does not hardcode element names): `get-element(name, "html")` (token refs resolve against `/shared/brand-book/tokens/*`); `rendering_missing`/`found:false` on an element → native default for that one element, keep going.
4. **Personal-element precedence.** Read LOCAL `members/{member_hash}/brand-book/personal/elements/{slug}.json` (native tools): if registry `provider_config.brand_usage == "optional"`, personal overrides org; if `"required"`, org wins and personal applies only to element types the org hasn't defined.
5. **Item set.** Build from the Step 1–4 rows: each item = `{name, slug, tier, status, owner, access}`. Per-item slot values (`item-logo`) come from the instance's `branding/` when readable at the item's tier; otherwise the card uses initials. Name-only rows are included (floor) but carry no extra data.

**Failure rule:** any failure in steps 1–5 degrades to the Step 6b table with a specific notice — brand resolution never blocks or fails the listing.

### Step 6a: Render — branded interactive directory (default in Cowork)

When Step 5 produced a template AND the surface is Cowork: compose the interactive directory from the template's `sections` — its header element over a per-item grid element (per the template's `data_binding`), one rendered per item, each showing the item's name, tier, status, and the template's `marker_fills` buttons.

Wire each abstract action handle from the template's `action_contract` to this collection's flow (this is the consumer mapping the brand book intentionally leaves open):

| Template handle | Scope | This collection runs |
|---|---|---|
| `add` | collection | **create-client** (the add-client interview) |
| `view` | item (`item_key` = slug) | **view-client** for that slug |
| `edit` | item (`item_key` = slug) | **edit-client** for that slug |
| `archive` | item, `confirm` | **delete-client** `mode: archive` for that slug (soft; floor record survives) |

In Cowork the buttons emit their handle (+ slug for item scope) back into the session; on click, run the mapped task per its own definition (including its own confirmation gate — `archive` is confirm-gated by delete-client). After a flow completes, re-render the directory so the change shows. Respect the floor: never expose data on name-only cards; **`view` is offered on every card** (read-only — on a name-only client view-client shows just the floor record plus the share hint), while `edit`/`archive` are shown only when the caller can act (e.g. archive on a private client they don't own is omitted, or defers to the mapped task's own authority check).

### Step 6b: Render — markdown table (fallback / on request)

When there is no brand template, the surface isn't Cowork, or the member asks for "a table":

```
| Slug | Name | Owner | Access | Status | Template | Created |
|---|---|---|---|---|---|---|
| acme-pharma | Acme Pharma | Bill | org-public (edit) | active | pharma-client v3 | 2026-05-13 |
| globex | Globex | testproduction | shared with me (read) | active | pharma-client v3 | 2026-06-01 |
| falcon | Project Falcon | jeff | name-only | active | pharma-client v3 | 2026-06-02 |
```

Next-action hints: view (`@ai:view-client {slug}`), create, and for name-only rows: *"ask {owner} to share it (`@ai:grant-permission` is owner-run)"*.

### Step 7: Export on request (standalone artifact)

If the member asks to export/save the directory: render the same composition to the requested format from the template's `surfaces.export_formats` — `html` (self-contained styled page), `pdf`, `docx`, or `markdown`. Use the brand-book element renderings for that format; action buttons are omitted in static formats (per `action-button`'s static rule) unless an interactive HTML export is explicitly requested. Save the file as an artifact and link it. Exports never write to the commons and never change client data.

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
