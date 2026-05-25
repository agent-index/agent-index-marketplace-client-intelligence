# Changelog

## [1.1.1] — <RELEASE_DATE> — companion to core 3.7.4

### Documentation

- **V1 data-visibility-floor claim corrected.** The 1.1.0 release notes claimed the limitation was "RESOLVED" by `inherit: false` activation. In practice, `inheritedPermissionsDisabled` only blocks immediate-parent inheritance; ancestor grants further up the tree still propagate. Bug `20260522-8d20ea22-3` documented the gap.
- `create-client.md` § Data visibility floor: rewritten to mark as "partially resolved" and document two operational patterns for confidential engagements (codename pattern for moderate confidentiality; empty-shell + off-platform for highest sensitivity).
- `grant-permission.md` — added a `## V1 Limitations` section that didn't previously exist; same correction text plus a cross-reference to `create-client.md` for the operational pattern detail.
- `grant-permission.md` — incidentally completed a pre-existing truncated line in Edge Cases (post-state verification failure handling). Pre-existing issue from the 1.1.0 release; fixed in passing since the file was being touched anyway.

### Notes

- No spec mechanic changes; documentation-only. Task versions for `create-client` and `grant-permission` stay at 1.1.0; manifests get a `collection_version` bump to 1.1.1 (mechanical).
- Real design fix tracked as core-improvements idea `data-visibility-floor-ancestor-leak`. Two design options (apply override higher in the tree; or restructure all-members grant location) deferred to 3.8.0+ alongside broader access-control project work.
- Companion releases: agent-index-core 3.7.4 fixes other 3.7.3 follow-ups (Node-helper removal, publish-updates writeback regression, non-admin onboarding). agent-index-filesystem-gdrive 2.4.0 for the non-admin onboarding adapter fixes.

---

## [1.1.0] — 2026-05-20 — companion to core 3.7.3, gdrive 2.3.0

### Fixed

- **V1 data-visibility-floor limitation resolved.** Previously documented in V1 release notes: per-instance ACLs applied by `create-client` and `grant-permission` were additive on top of the inherited all-members Writer grant from `/shared/client-intelligence/instances/`, meaning every collection member retained Writer access on every new instance regardless of the intended grant set. The visibility floor on instance names worked (public-index gating); the floor on instance contents did not. Codenames were the only confidentiality mechanism that worked in V1. Now resolved end-to-end across three collections:
  - agent-index-core 3.7.3 added `inherit: boolean` to the permission-change-helper spec format (v1.1; closes idea `helper-spec-needs-inherit-passthrough` sections 1–3).
  - agent-index-filesystem-gdrive 2.3.0 actually implements `inherit: false` via `inheritedPermissionsDisabled: true` on the file resource (closes the adapter portion of the same idea; requires organizer role on the Shared Drive).
  - **This release activates `inherit: false` in `create-client` and `grant-permission` callers.** Per-instance shares now correctly narrow below parent-folder inheritance.

### Changed

- **`create-client` 1.0.0 → 1.1.0** — Step 13's helper spec is now v1.1 with `inherit: false` on every per-instance share operation. Purpose string updated to mention "parent-inheritance override." Limitations section "Data visibility floor" rewritten to mark the limitation resolved. The codename pattern remains useful for engagements where the *existence* of the instance is sensitive (instance names are still discoverable via public-index) but is no longer required for data confidentiality.
- **`grant-permission` 1.0.0 → 1.1.0** — Step 7's helper spec is now v1.1 with `inherit: false`. Same activation mechanics as `create-client`. Limitations section updated.

### Notes

- **Versions:** collection 1.0.0 → 1.1.0. `create-client` task 1.0.0 → 1.1.0. `grant-permission` task 1.0.0 → 1.1.0. All API manifests' `collection_version` bumped to 1.1.0. Other tasks unchanged at their original versions but their manifest `collection_version` field also bumped.
- **Permission requirement:** activating `inherit: false` requires the user who clicks Accept on the helper review page to have `organizer` role on the Shared Drive (or `owner` on My Drive). Org admins always have this; non-admin members will see a clean `AccessDeniedError` from the adapter with an actionable message. The current admin-driven instance creation flow works transparently; this constraint becomes meaningful if/when non-admin instance creation is enabled in a future release.
- **Unshare ops unchanged:** `revoke-permission` and `remove-admin` use `op: "unshare"` which doesn't carry the `inherit` field. Removing an explicit grant does NOT re-enable parent-folder inheritance (the file's `inheritedPermissionsDisabled` flag persists across share/unshare cycles by design). If a workflow ever needs to "fully restore parent inheritance," that's a separate semantic — out of scope for 1.1.0.
- **`add-admin` unchanged:** the admin role is granted on `templates/` and `config/`, whose parent `/shared/client-intelligence/` has only `reader` for all-members. Sharing those subfolders with admins narrows correctly under the default semantic; no `inherit: false` needed.

---

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
