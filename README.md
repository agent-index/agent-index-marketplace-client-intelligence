# Client Intelligence Collection

The system's primary repository of client-level knowledge. Members create, manage, and remove records of the clients an organization works with. Each client engaged with is represented by a client instance.

## Status

v1.0.0 is in active development. The collection's design is settled across three documents in the `agency-workflow` project:

- `client-intelligence-collection` — the design (what & why)
- `client-intelligence-storage-layout` — the on-disk shape (where)
- `client-intelligence-task-surface` — the user actions (how)

Implementation is tracked in the `client-intelligence-collection` project. v1.0.0 ships 15 user-facing tasks plus install-time setup. See `CHANGELOG.md` for in-flight changes.

## Core Concepts

**Templates.** Admin-authored definitions of what fields a client instance carries. Each field is mandatory or optional. Templates are versioned; admins choose Migrate or No-impact at the moment of each edit.

**Client instances.** Per-client records, created from a template, optionally extended with member-added fields. Each instance has its own folder on the remote filesystem with its own ACL.

**Per-instance permissions.** Three independent permissions (View, Edit, Delete) granted per-(member, instance). Initial grants compose three mechanisms — creator default (creator unconditionally gets all three), collection-level default policy (admin-configured), and per-creation explicit grants.

**Visibility floor.** A client's name is universally visible to all collection members regardless of view permission; only the data is gated. Codenames are the operational pattern for sensitive engagements.

**Audit.** Permission events live in Drive Activity, queried via the access-control adapter. Data events (field edits, additions, removals, soft/hard deletes) are recorded in append-only per-instance changelogs.

## Included Capabilities

v1.0.0 includes 15 tasks plus 1 tutorial skill, across five areas:

**Template management:**
- `create-template` (admin) — Author a new template.
- `edit-template` (admin) — Edit a template; choose Migrate or No-impact at the edit moment.
- `list-templates` (any member) — Enumerate templates available for client creation.
- `view-template` (any member) — Show a template's current version and version history.

**Client instance management (any member):**
- `create-client` — Create a new client instance from a template.
- `view-client` — Read a client's data and recent changelog.
- `edit-client` — Modify field values; add/remove extension fields.
- `delete-client` — Soft or hard delete a client.
- `list-clients` — List all client names (data hidden where view perm absent — visibility floor).

**Permission management (current grantees + admins):**
- `grant-permission` — Grant view/edit/delete on a client to a member.
- `revoke-permission` — Revoke view/edit/delete from a (member, client) pair.
- `view-permissions` — List who has what permissions on a client.

**Admin lifecycle (admin-only):**
- `add-admin` — Grant admin role.
- `remove-admin` — Revoke admin role.
- `edit-default-permissions` — Modify the collection-level default permission policy.

**Tutorial:**
- `client-intelligence-tutorial` (skill, any member) — Conversational guided tour and question-answering about how the collection works. Say `@ai:client-intelligence-tutorial` for the seven-topic tour, or ask a specific question to get a scoped answer.

Plus the install-time setup workflow (documented in `setup/collection-setup.md`) that bootstraps the collection on a fresh org install.

## How It Works

The collection lives at `/shared/client-intelligence/` on the remote filesystem with the following layout:

```
/shared/client-intelligence/
├── collection.json              # collection metadata
├── config/
│   └── default-permissions.json # collection-level default permission policy
├── templates/                   # writers of this folder = admins
│   ├── _changelog.json
│   └── {template-slug}/
│       ├── template.json
│       └── versions/vN.json
├── public-index/                # visibility-floor index (universally readable)
│   └── instances/
│       └── {instance-slug}.json
└── instances/                   # per-instance ACL'd data
    └── {instance-slug}/
        ├── instance.json
        └── changelog.json
```

Admin role is derived from the filesystem — a collection admin is any member with Drive Editor access to `templates/`. Tasks do not pre-check admin status; they attempt the operation and translate permission-denied responses into clear caller errors.

## Prerequisites

- agent-index-core 3.1.0 or later (for access-control adapter contract 2.0.0+)
- Filesystem adapter with contract 2.0.0 support (`aifs_share`, `aifs_unshare`, `aifs_get_permissions`, `aifs_search`, revision-aware `aifs_write`)
- An org-level all-members Google Group defined in `org-config.json` (`remote_filesystem.connection.all_members_group`)

## Lifecycle

1. Org admin installs the collection via the standard `install-collection` flow.
2. The install-time setup task collects bootstrap admin(s), default permission policy preset, and example-template choice; then creates the folder tree, applies initial ACLs, seeds config, and ships the example template.
3. Day-to-day use: admins author templates; members create clients from templates, share with collaborators, edit, and (eventually) archive or delete.
4. New admins are granted via `@ai:add-admin` (which shares `templates/` and `config/` with them). Removing an admin is the reverse.

## Version History

See `CHANGELOG.md`.
