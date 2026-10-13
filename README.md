# Client Intelligence Collection

The system's primary repository of client-level knowledge. Members create, manage, and remove records of the clients an organization works with. Each client engaged with is represented by a client instance.

## Status

Current release line: 2.x (see `collection.json` for the exact version). 2.0.0 replaced the 1.x per-instance ACL model with the two-tier visibility model described below; 2.4.0 moved every access change onto the core permission-changes procedure. The original design documents live in the `agency-workflow` project (`client-intelligence-collection`, `client-intelligence-storage-layout`, `client-intelligence-task-surface`); the 2.0 model is design 07. See `CHANGELOG.md` for release history and `upgrade/1-to-2.md` for upgrading a 1.x org.

## Core Concepts

**Templates.** Admin-authored definitions of what fields a client instance carries. Each field is mandatory or optional. Templates are versioned; admins choose Migrate or No-impact at the moment of each edit.

**Client instances.** Per-client records, created from a template, optionally extended with member-added fields. Each client lives in one of two tiers:

- **Private** (the default) — stored in the owner's own My Drive (`id:{member_folder_id}/clients/{slug}/`). Only the owner has access until they share it. Sharing uses the org-wide vocabulary: *share with X* = X can read; *make X a collaborator* = X can read and write. The owner is the only grantor; every grant or removal goes through the core permission-changes procedure (the owner confirms the exact change, it is applied under their own credentials, then verified). People outside the org can be shared with; they're flagged in the confirmation table and open the client from their storage app's "Shared with me".
- **Org-public** — stored in the org commons (`/shared/client-intelligence/instances/{slug}/`) under uniform member write access, with mandatory changelog attribution. The org keeps custody regardless of who comes or goes.

`transition-client` moves a client between tiers. There are no per-instance ACLs and no separate View/Edit/Delete grants (the 1.x model is retired).

**Visibility floor.** Every client has a pointer record in `public-index/instances/` carrying its name, owner, tier, and status, so every client's name is org-discoverable regardless of tier; only the data is gated. Codenames are the operational pattern for sensitive engagements.

**Audit.** Permission events live in the backend's own activity log (Drive Activity). Data events (field edits, additions, removals, archival) are recorded in append-only per-client changelogs. Floor records are permanent history; archival is the default lifecycle verb.

## Included Capabilities

17 tasks plus 1 tutorial skill, across six areas:

**Template management:**
- `create-template` (admin) — Author a new template.
- `edit-template` (admin) — Edit a template; choose Migrate or No-impact at the edit moment.
- `list-templates` (any member) — Enumerate templates available for client creation.
- `view-template` (any member) — Show a template's current version and version history.

**Client instance management (any member):**
- `create-client` — Create a new client from a template; choose private or org-public.
- `view-client` — Read a client's data and recent changelog.
- `edit-client` — Modify field values; add/remove extension fields.
- `delete-client` — Archive (default) or remove a client; the floor record always survives.
- `list-clients` — List every client's name, owner, tier, and status (data only where you have access — visibility floor).
- `transition-client` (owner) — Move a client between the private and org-public tiers.
- `client-brief` (any member with access) — Shareable client brief document; brand-styled when the org registers a brand-book provider. Instance `branding/` supplies the client logo/name as slot values.

**Sharing private clients (owner-only):**
- `grant-permission` — Share a private client (read) or make someone a collaborator (read+write).
- `revoke-permission` — Remove someone's access to a private client.
- `view-permissions` (any member) — Show who can access a client, cross-checked against live backend state.

**Admin lifecycle (collection admins):**
- `add-admin` — Grant admin role (Editor on `templates/` and `config/`).
- `remove-admin` — Revoke admin role (never the last admin).
- `edit-default-permissions` — Set the default answer to create-client's visibility question (`private_first` / `ask` / `org_public_first`).

**Tutorial:**
- `client-intelligence-tutorial` (skill, any member) — Conversational guided tour and question-answering about how the collection works. Say `@ai:client-intelligence-tutorial` for the seven-topic tour, or ask a specific question to get a scoped answer.

Plus the install-time setup workflow (documented in `setup/collection-setup.md`) that bootstraps the collection on a fresh org install.

## How It Works

The collection lives at `/shared/client-intelligence/` on the remote filesystem with the following layout:

```
/shared/client-intelligence/
├── collection.json              # collection metadata
├── collection-state.json        # install record (version, admins, default visibility)
├── config/                      # writers of this folder = admins (1.x default-permissions.json retired)
├── templates/                   # writers of this folder = admins
│   ├── _changelog.json
│   └── {template-slug}/
│       ├── template.json
│       └── versions/vN.json
├── public-index/                # visibility floor: one pointer per client (all members write)
│   └── instances/
│       └── {instance-slug}.json
└── instances/                   # org-public tier commons (uniform member access)
    └── {instance-slug}/
        ├── instance.json
        └── changelog.json
```

Private clients live outside this tree, in each owner's own My Drive at `id:{member_folder_id}/clients/{slug}/`; their pointer records the folder ID (and the owner's `item_drive_id` for cross-drive opening).

Admin role is derived from the filesystem — a collection admin is any member with Drive Editor access to `templates/` and `config/`; they need not be org admins. The backend enforces it: admin tasks don't pre-check the role, and a non-admin's write or share is refused and translated into a clear error.

## Prerequisites

- agent-index-core 3.32.0 or later (the core permission-changes procedure applies every access change; My Drive member spaces)
- Filesystem adapter with contract 2.0.0 support and ID-anchor addressing (gdrive adapter 2.5.0+): `aifs_get_permissions`, `aifs_search`, revision-aware `aifs_write`. Members never call the sharing operations directly — the core procedure does, after the person confirms.
- An org-level all-members Google Group defined in `org-config.json` (`remote_filesystem.connection.all_members_group`)

## Lifecycle

1. Org admin installs the collection via the standard `install-collection` flow.
2. The install-time setup collects bootstrap admin(s), the default visibility prompt, and example-template choice; then creates the folder tree, seeds `collection-state.json`, ships the example template, and applies the folder grants through the core permission-changes procedure (the installing admin confirms the exact table).
3. Day-to-day use: admins author templates; members create clients (private or org-public), share private ones with collaborators, edit, move them between tiers, and archive them.
4. New admins are granted via `@ai:add-admin` (which shares `templates/` and `config/` with them). Removing an admin is the reverse.
5. A 1.x org upgrades via `upgrade/1-to-2.md` — the gated-instance review must finish before the `instances/` grant is applied.

## Version History

See `CHANGELOG.md`.
