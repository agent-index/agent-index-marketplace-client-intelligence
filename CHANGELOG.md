# Changelog

## [2.4.1] — 2026-10-13 — Fixes from the dev-1 live test (Dev Two)

### Fixed
- **`view-permissions` 2.0.2 no longer collides with core's (`civiewpermsclash`, finding 24).** Its alias is `@ai:view-client-permissions`, and its triggers name the client ("view client permissions", "who has access to client", "who can see client", "permissions on client"). Core 4.0.0 apply-updates removes the retired generic phrases from existing members' routing, and org-setup's upgrade flow replaces the old alias in their member-index.
- **`remove-admin` counts inherited admins (`ciadmincount`, finding 19).** Drive Managers who hold Editor through inheritance count toward the last-admin guard; the all-members group never does. (Staged on dev-1 under 2.4.0.)

### License
- Proprietary, as before: `LICENSE` added and the `collection.json` `license` field set to `proprietary` (it said `open`).

## [2.4.0] — 2026-10-02 — Client access through the core permission-changes procedure

**MINOR — requires agent-index-core 3.32.0 (and agent-index-marketplace 2.22.0 for collection provisioning).** Part of retiring the permission-change-helper (design record: `/shared/projects/core-improvements/artifacts/retire-permission-helper-*.md`).

### Changed
- **`create-client` 2.4.0, `grant-permission` 2.4.0, `revoke-permission` 2.1.0, `transition-client` 2.1.0** — every grant and removal goes through core `internal/permission-changes.md`. The owner sees the exact table and confirms it (removals one at a time), and each change is verified. Pointer scope lists only satisfied grants. After a revoke, scope is recomputed from a fresh `aifs_get_permissions` read.
- **Recipients outside the org are allowed** (by email; flagged **outside your org**) for per-client sharing. Admin role (`add-admin`) is still limited to org members.
- **`add-admin` 2.1.0, `remove-admin` 2.1.0** — the two-folder admin grant/removal goes through the procedure in `all_or_nothing` mode. Declining either row applies neither (core 3.32.0). remove-admin's separate yes/no step is folded into the procedure's per-removal confirmation. The last-admin guardrail is unchanged.
- **`remove-admin` last-admin guardrail counts inherited admins (`ciadmincount`, found in the 3.32.0 live test).** Drive Managers who hold writer on `templates/` through inheritance now count as remaining admins. Removing an added admin is no longer refused because the bootstrap admin's grant was skipped as already in effect. The all-members group never counts.
- **Collection setup § 8–9** — bootstrap ACLs go through the procedure as one admin-confirmed table. The repair path re-runs it (it skips what's in place).
- **Task setups 1.1.0: `add-admin-setup`, `remove-admin-setup`, `create-client-setup`, `grant-permission-setup`, `revoke-permission-setup`** — helper-binary pre-checks removed; stale migration notes replaced with a 1.1.0 note.
- **`add-admin` / `remove-admin` use `confirmer: "collection_admin"`** — client-intelligence admins (Editor on `templates/` and `config/`) need not be org admins; the backend enforces the role. Outputs name the `sharing_identity`.
- **`add-admin`** — "already an admin" is decided by `nothing_to_change` before the `complete` branch; any other failure relays the procedure's summary and reports nothing applied.
- **`remove-admin`** — self-removal is a notice (the per-row confirmations are the only confirmation); a departed member outside the registry can be picked from the current writers (`source: "current_state"`); the admin test reads the unfiltered permissions of both folders (the filtered set is only the last-admin count), so inherited-only and stray `config/` grants are handled; "no longer an admin" is printed only after a fresh re-read shows no Editor remains; a non-access-denied failure recommends re-running to finish (the revert is the alternative).
- **`grant-permission`** — no early exit on "already granted per pointer scope": the procedure decides (`skipped_noop`) and the pointer is still reconciled from a fresh `aifs_get_permissions` read (and `item_drive_id` backfilled). Outside-org recipients are told to use "Shared with me".
- **`revoke-permission`** — an inherited-only target is told the access is inherited and can't be removed here (not "doesn't have access"); pointer reconciliation after a failed removal is a separate fresh-read overwrite on the owner's request.
- **`create-client`, `grant-permission`, `revoke-permission`, `transition-client`** — every change carries a `reason`.
- **`transition-client`** — private → org-public cleanup captures the old `folder_id` in Step 1, reads its grants fresh, asks which to remove before the table, and sends one `unshare` per subject (`source: "current_state"`); org-public → private writes `item_drive_id` into the new pointer (C.1.3 crossdriveread).
- **Collection setup 1.1.0** — sources declared per row (`org_config` for the all-members group, `member_request` for bootstrap admins); `bootstrap_admins` is written to `collection-state.json` only after § 8 completes; `version` fields templated from `collection.json`; org-config and the members registry read by id anchor; a 1.x `collection-state.json` halts and routes to `upgrade/1-to-2.md`.
- **`client-intelligence-tutorial` 2.0.1, `view-permissions` 2.0.1** — wording: confirm the exact change in conversation (no review page); pointer repaired "after verified changes".
- **README** refreshed for the 2.x tier model; requires core 3.32.0+.

### Fixed
- **`collaborative-acls.json` used `grants[]` / `purpose`, which install-collection ignored**, so its declared grants were never provisioned by Step 5.5 (the setup's own § 8 masked it). It now uses `acls[]` / `rationale` and declares `"requires_upgrade_review": ["1-to-2"]`, which install-collection reads to ask whether the 1-to-2 admin-half review is done before applying these grants on an upgrade from 1.x. install-collection 2.4.0 also accepts the old key.
- **Manifest drift** — `external_dependencies`, `reads_from` / `writes_to` synced to frontmatter for create-client, grant-permission, revoke-permission, view-client, list-clients, edit-client, delete-client, view-permissions, edit-default-permissions; the spurious `view-client` task dependency is dropped from edit-client and delete-client manifests (neither workflow invokes it).

### Removed
- Dependency on the `permission-change-helper` skill and binary (frontmatter, manifests, `collection.json` external dependency).
- **1.x → 2.x upgrade no longer loops:** `upgrade/1-to-2.md` step 4a records `"1-to-2"` in org-config `upgrade_reviews_confirmed` once the gated-instance review is done. The setup's 1.x gate halts only while that marker is missing.
- **add-admin / remove-admin:** access-denied messages and the V1 note say plainly that changing sharing on these Shared-Drive folders needs drive-Manager rights (F12).

## [2.3.0] — 2026-06-28 — Release C.1.3: crossdriveread

### Fixed
- **`crossdriveread`** — `create-client` and `grant-permission` capture `item_drive_id` from `aifs_stat`'s `drive_id` into the client pointer's `location`; `view-client`/`edit-client` open private clients shared to them via the cross-drive anchor `id:{item_drive_id}:{folder_id}/...` (bare-anchor fallback). Fixes opening a client instance held in another member's personal OneDrive. Requires onedrive adapter 2.3.0+; harmless on gdrive.

## [2.2.0] — 2026-06-17 — branded interactive client views (directory + profile)

### Added

- **`list-clients` branded interactive view (default in Cowork).** When a brand-book provider supplies the `client-list` template, `list-clients` now composes an interactive directory from the template's elements (per-item cards under a header) instead of only the markdown table. The template's action buttons are wired to live flows: page-level **Add client** → `create-client`; per-card **View** → `view-client`, **Edit** → `edit-client`, **Archive** → `delete-client` (`mode: archive`, soft; the floor record survives). Buttons run the mapped task with its own confirmation gate and authority check; the directory re-renders after a flow completes.
- **Standalone export.** On request, `list-clients` renders the directory to a standalone artifact — HTML / PDF / Word / markdown — from the brand-book renderings (action buttons omitted in static formats). Exports are local; they never write the commons or change client data. `produces_artifacts` flips to `true`.
- **`view-client` branded interactive profile (default in Cowork).** When a brand-book provider supplies the `client-profile` template, `view-client` composes a branded profile from the template's elements (a header over detail/field/activity sections) with record actions wired to flows: **Edit** → `edit-client`; **Manage access** → `view-permissions` / `grant-permission` / `revoke-permission`; **Generate brief** → `client-brief`; **Change visibility** → `transition-client`; **Archive** → `delete-client` (`mode: archive`). Actions are authority/tier-gated; a name-only floor view shows no record actions. Falls back to the plain record when no provider is present, and can export the profile (HTML/PDF/Word/markdown) on request. Brand resolution inlined per int4. `produces_artifacts` → `true`.
- Brand resolution is **inlined** in `list-clients` Step 5 (id-anchored provider base, registry-sourced `brand_book_version`, personal-element precedence, degrade-never-block), consistent with the int4 fix in `client-brief` 2.1.1 — members can't read `/internal/` at runtime.

### Consumer mapping (brand-book convention v1.1)

- Implements the brand-book "action markers" convention: the `client-list` template declares neutral action handles (`add` / `edit` / `archive`) in its `action_contract`; this collection maps each to one of its own tasks. The brand book never names a client-intelligence task. Requires brand-book ≥ 1.1.0 for the `client-list` template + interactive-template schema fields (still `required: false`, `fallback: skip_with_notice` — older/absent brand books degrade to the table).


## [2.1.1] — 2026-06-08 — inline brand resolution (int4)

### Changed

- `client-brief` Step 3 now inlines the brand-resolution logic (provider lookup via the registry, id-anchored provider read base, personal-element precedence, instance-branding slot values, degrade-never-block) instead of referencing `/internal/resolve-brand.md`. Members cannot read collection `/internal/` files at runtime (not synced locally; not path-resolvable on the org remote — bug 20260608 int4), so the referenced mechanism never executed. `/internal/resolve-brand.md` remains as authoring documentation. Also adopts id-anchored provider addressing (core 3.10.1) and registry-sourced `brand_book_version` (avoids the db13 name-resolution path).

## [2.1.0] — 2026-06-07 — first brand-book consumer (companion to core 3.10.0 / marketplace 2.10.0 / brand-book 1.0.0)

### Added

- **`requires[]`: brand-book** (`required: false`, `fallback: skip_with_notice`) — the first consumer declaration in agent-index. Resolution per the new `/internal/resolve-brand.md` (single-provider auto-bind; personal-element precedence; slot values from instance context; degrade-never-block).
- **`client-brief` task** — shareable client brief document (markdown/docx) from an instance, tier-mechanical access, LOCAL output, brand-styled when a provider is registered.
- **Instance `branding/` context** — `edit-client` manages `client_display_name` + `client-logo` INSIDE the instance (structural tier inheritance); these are SLOT VALUES for org-branded templates — the brand book never holds client assets.

### Requires Admin Attention

- Requires core 3.10.0 + marketplace 2.10.0. Install/register brand-book 1.0.0 to activate styling; without it everything works unbranded.

## [2.0.0] — 2026-06-05 — two-tier visibility model (MAJOR, breaking)

### Requires Admin Attention (read BEFORE provisioning)

**The upgrade ships ACL and setup-interview changes** (`collaborative-acls.json` is new;
`clients_default_visibility` replaces `default_permission_policy_preset`) — upgrade-collection
Step 6.5 will flag provisioning. **ORDERING RULE: run the gated-instance review in
`upgrade/1-to-2.md` BEFORE `@ai:install-collection` applies the `instances/` all@ writer grant.**
1.x instances may carry per-instance gating that the uniform grant would expose to the whole org.

### Why a MAJOR

The 1.x model had the *creator* apply per-instance Drive ACLs (with `inherit:false`) on folders
under `/shared` — Drive permits folder-sharing on Shared Drives only to drive Managers
(finding **F12**), so every non-admin `create-client` failed at the ACL step, and `inherit:false`
additionally required organizer. The per-instance ACL model was unimplementable for its primary
users. 2.0 replaces it with the two access models proven in the bug-reports and strategy reworks.

### Changed

- **Two visibility tiers.** PRIVATE (default): the instance lives in the member's own My Drive
  (`id:{member_folder_id}/clients/{slug}/`, core 3.9.0 member spaces); the owner shares per-person
  (share=reader, collaborator=writer) via permission-change-helper with their own Accept,
  hard-gated on verified outcomes (strategy 1.1.3 precedent). ORG-PUBLIC: the instance lives in
  the commons (`/shared/client-intelligence/instances/`) under uniform `all@` writer access with
  mandatory changelog attribution on every edit (bug-reports precedent). No per-instance ACLs
  exist anywhere — if a client needs gating, it belongs in the private tier.
- **Universal visibility floor → pointer index.** `public-index/instances/{slug}.json` becomes a
  full pointer record (name/owner/owner_hash/template/status/scope/location/owner_departed),
  written at creation for BOTH tiers. Duplicate detection and relationship discovery preserved;
  the 1.x ancestor-leak limitation is structurally dissolved (private data has no org ancestors).
- **All 16 capabilities reworked or bumped to 2.0.0**; new task **transition-client** moves
  clients between tiers (copy → pointer flip → source marked; commons originals become archived
  stubs — going private does not un-publish). `grant/revoke-permission` are owner-only,
  private-tier-only. `edit-default-permissions` now sets the org's creation-prompt default
  (`clients_default_visibility`: private_first | ask | org_public_first) — no ACL policy exists.
  `delete-client`: archival is the default verb; floor records are permanent; true deletion only
  for owners in their own Drive.
- **`config/default-permissions.json` retired** (overwritten `{"retired": "2.0.0"}` on upgrade).

### Added

- `collaborative-acls.json` — all@ reader on the collection root, all@ writer on `public-index/`
  and `instances/` (same set the install bootstrap applies; now declaratively provisionable).
- `upgrade/1-to-2.md` — REQUIRED MAJOR migration: admin inventory + gated-instance review +
  pointer-index rebuild + policy-file retirement + provisioning order; member half is optional
  (transition-client for keep-gated instances).

### Requirements

- agent-index-core **3.9.0+**, gdrive adapter **2.5.0+**, permission-helper-go **0.4.1+**.
- 1.x line: `eol_date` set 90 days from this release (2026-09-03) per EOL policy.

---

## [1.1.1] — 2026-10-13 — companion to core 3.7.4

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
