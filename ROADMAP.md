# Client Intelligence — Roadmap

## Current State

v1.0.0 is in active development. Design is settled in the three design documents in the `agency-workflow` project (design, storage layout, task surface). Implementation is tracked phase-by-phase in the `client-intelligence-collection` project — install-time setup → template lifecycle → instance lifecycle → permission management → admin lifecycle → preflight & release.

The collection is the first entry in agency-workflow's collection-by-collection design series. Each collection is defined in isolation before inter-collection mechanics (pub/sub wiring, cross-collection references) are layered on. Those mechanics are explicitly out of scope for v1.0.0.

## V1 Scope

Fifteen user-facing tasks plus install-time setup. Full enumeration in `README.md` and `CHANGELOG.md`.

## Known Limitations (V1)

- **No permission-history audit query.** The "who has access to this client and when did they get it" query is blocked on the access-control project's `aifs_get_audit` (or equivalent) adapter operation, which is still being scoped. Day-to-day permission state queries via `view-permissions` are not blocked; only the historical audit is.

- **Free-text fields only.** V1 templates have no field-type system or validation rules. All fields are free-text. Typed fields and validation will be introduced when specific templates need them.

- **Name-prefix search only.** `list-clients` supports name-prefix filtering. Faceted search and full-text search across instance data are deferred.

- **No bulk import/export.** Clients are created one at a time through the standard creation flow. Migration from a spreadsheet or another CRM at install time is not supported in V1.

- **No linked external data references.** Instances carry template-defined fields plus member-added extension fields only. Integration with external data sources (CRMs, marketing platforms, billing systems) is post-V1.

- **Fixed visibility floor.** Instance names are universally visible; data is gated by view permission. No per-template or per-instance configuration of this rule in V1.

- **Last-write-wins concurrency.** V1 uses revision-aware writes; on conflict the second writer fails and must retry. Richer conflict UX (merge views, field-level locks) is deferred.

- **No uninstall task.** Removing the collection in V1 means manually deleting `/shared/client-intelligence/` and editing org-config.

## Wishlist (post-V1)

- `view-client-audit` task that combines local data changelog with FS-sourced permission-event history. Unblocks when access-control ships `aifs_get_audit`.
- Typed fields and validation rules within templates.
- Faceted and full-text search.
- Bulk import / export.
- Linked external data references.
- Per-template visibility-floor configuration.
- Bulk permission operations ("revoke Sarah from all clients", "grant view on all Pharma clients to Bill").
- Uninstall task.
- Cross-collection wiring (pub/sub) — this is a separate scope blocked on the broader agency-workflow architecture decisions, not on this collection alone.

## Known Bugs

None yet — collection has not shipped.
