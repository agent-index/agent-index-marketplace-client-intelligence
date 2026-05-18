# Changelog

## [1.0.0] — 2026-05-15

Initial release. First collection of the agency-workflow collection-by-collection design series. Built against three design documents in the `agency-workflow` project (design, storage layout, task surface) plus a V1 tech design in the `client-intelligence-collection` build project. Implementation proceeded through six phases (scaffold → templates → instances → permissions → admin → preflight & release) with end-to-end install verification before publish.

Ships **15 user-facing tasks plus 1 tutorial skill (16 capabilities total)**, plus the install-time setup workflow that bootstraps the collection on a fresh org install.

### Added

- Install-time setup workflow (documented in `setup/collection-setup.md`) that bootstraps `/shared/client-intelligence/`, creates the folder tree, applies initial ACLs via the permission-change-helper (all-members Google Group reader on the root and instances/public-index folders; bootstrap admins receive Editor on `templates/` and `config/` via the helper-applied share), seeds `config/default-permissions.json` and `templates/_changelog.json`, and optionally installs an example template.

- Template management:
  - `create-template` (admin) — Author a new template with mandatory + optional fields.
  - `edit-template` (admin) — Edit an existing template; choose Migrate or No-impact migration path per edit.
  - `list-templates` (any member) — Enumerate templates.
  - `view-template` (any member) — Show a template's current version + version history.

- Client instance management:
  - `create-client` (any member) — Create a client from a template; computes initial permission grant as the union of creator default + collection-level default policy + per-creation explicit grants.
  - `view-client` (view perm) — Read a client's data and tail of its changelog.
  - `edit-client` (edit perm) — Modify field values; add/remove extension fields.
  - `delete-client` (delete perm) — Soft or hard delete a client.
  - `list-clients` (any member) — List all clients; data hidden where view perm absent (visibility floor).

- Permission management:
  - `grant-permission` (current grantee or admin) — Grant view/edit/delete on a client.
  - `revoke-permission` (current grantee or admin) — Revoke from a (member, client) pair.
  - `view-permissions` (any member with view) — List who has what on a client.

- Admin lifecycle:
  - `add-admin` (admin) — Grant admin role by sharing `templates/` and `config/`.
  - `remove-admin` (admin) — Revoke admin role; last-admin guardrail enforced.
  - `edit-default-permissions` (admin) — Modify the collection-level default permission policy.

- Tutorial:
  - `client-intelligence-tutorial` (skill, any member) — Conversational guided tour of the collection plus question-answering mode. Seven topics covering what the collection does, templates, the create/view/edit/delete client flow, the permission-helper friction, the visibility floor (with V1 limitation called out), deletion modes, and admin lifecycle. Reads live collection state at invocation so examples reflect the actual install.

### Design notes

- Filesystem is the single authority for permission state and permission audit. The collection maintains no parallel permission state and no parallel permission-event log.
- Admin role is derived from the filesystem — admins are members with Drive Editor access to `templates/`. Tasks use try-and-catch on operations rather than pre-checking authority.
- Per-instance permissions are enforced by Drive ACLs via `aifs_share` / `aifs_unshare`. View → Drive Reader; Edit and Delete → Drive Writer (delete is task-layer enforcement).
- Visibility floor is implemented as a public/private split: `public-index/instances/{slug}.json` carries name + status (universally readable); `instances/{slug}/instance.json` carries the data (gated by view perm).
- Enumeration uses `aifs_list` on `public-index/instances/` — no manifest.
- Concurrency uses revision-aware `aifs_write` (the `if_revision` parameter from the access-control adapter contract).
- All permission-modifying operations go through `agent-index-core`'s `permission-change-helper` skill. The agent never calls `aifs_share` / `aifs_unshare` / `aifs_transfer_ownership` directly; the helper opens a review page in the caller's browser and applies changes with the caller's own OAuth token after explicit Accept.

### Known V1 limitations

- **Data visibility floor depends on access-control Phase 5** — the helper spec v1.0 doesn't pass `inherit: false` through to `aifs_share`. Per-instance ACLs are additive on top of the inherited all-members Writer from `instances/`, so the visibility floor on **data** doesn't fully gate per-instance contents in V1. The visibility floor on **names** works correctly (public-index has its own ACL). The codename pattern is the operational data-confidentiality mechanism pre-Phase-5. Filed as `helper-spec-needs-inherit-passthrough` under the `builder-profile-adaptive-dev-process` umbrella in core-improvements.
- Implementation-time findings filed under the umbrella during V1 build: `policy-storage-resolved-vs-preset-pattern`, `static-assets-shipped-with-collection-source-pattern`, `dev-vs-install-process-conflation`, `helper-spec-needs-inherit-passthrough`, `preflight-missing-tutorial-check`.

### Deferred to post-V1

- `view-client-audit` task — depends on access-control's `aifs_get_audit` (or equivalent), still being scoped.
- Typed fields and per-field validation rules within templates.
- Faceted and full-text search across the client roster.
- Bulk import / export.
- Linked external data references.
- Per-template visibility-floor configuration.
- Richer concurrent-edit UX (merge view, field-level locks).
- Bulk permission operations.
