---
name: view-client
type: task
version: 1.0.0
collection: client-intelligence
description: Member-facing read task that displays a client's full data and recent changelog. Requires View permission on the instance folder; if the caller lacks it, surfaces the visibility-floor view (name only, from public-index) with a note explaining how to request access.
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

View Client shows one client in detail to a member with View permission on the instance folder. For members without view permission, the task gracefully degrades to the visibility-floor view: just the client name and the fact that the caller doesn't have data access, plus guidance on how to request it. Members can run this task as a discovery aid before deciding whether to ask for access.

Authority is filesystem-enforced. The task does not pre-check view permission; it attempts the read and translates the response.

### Inputs

- **`slug`** (required, interactive or argument) — the instance slug to view. If the member said *"show me Acme"*, resolve to the slug by checking `public-index/instances/{slug}.json` for the matching name (case-insensitive); if multiple match, ask the member to disambiguate by slug.
- **`changelog_tail`** (optional, integer, default `10`) — number of most-recent changelog entries to include.

### Outputs

A formatted view of the client surfaced to the caller. Two modes:

- **Full view** (caller has view permission): header with name + slug + template + created date; the full `template_fields` block; the full `extension_fields` block; the tail of the changelog.
- **Visibility-floor view** (caller lacks view permission): header with name + slug only; a note explaining the limited view; a suggestion to request access via `@ai:grant-permission`.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with the not-installed message.

### Step 2: Resolve slug

If the input is a slug, validate kebab-case. If the input is a name, run a public-index scan: `aifs_list("/shared/client-intelligence/public-index/instances/")`, read each `{candidate_slug}.json`, find name matches. Disambiguate as needed.

### Step 3: Confirm the client exists

`aifs_exists("/shared/client-intelligence/public-index/instances/{slug}.json")`. If false, halt with: *"No client with slug `{slug}` exists. Run `@ai:list-clients` to see all clients by slug."*

### Step 4: Read the public-index entry

`aifs_read("/shared/client-intelligence/public-index/instances/{slug}.json")`. Capture name, slug, template_slug, template_version, created, created_by, status.

### Step 5: Attempt to read the instance data

`aifs_read("/shared/client-intelligence/instances/{slug}/instance.json")`.

**Branch on the response:**

- **Success.** The caller has read access. Continue to Step 6 with full data.
- **Permission-denied.** The caller lacks view permission on the instance folder. Surface the visibility-floor view and exit:

  ```
  Client: {name}
  Slug: {slug}
  Template: {template_slug} v{template_version}
  Status: {active | archived}

  You don't have view permission on this client. Only the name and metadata above are visible to you.

  To request access, ask an existing grantee or an admin to run:
    @ai:grant-permission {slug} {your_email_or_member_hash}

  To see who currently has access, you'd need view permission yourself — but admins and existing grantees can run @ai:view-permissions {slug}.
  ```

- **Any other error.** Surface the error and halt. The public-index entry exists but the instance file is missing or unreadable — investigative pointer for an admin.

### Step 6: Read the changelog tail

`aifs_read("/shared/client-intelligence/instances/{slug}/changelog.json")`. Parse, take the last `changelog_tail` entries (default 10), most recent first.

If `aifs_read` fails on the changelog (rare — the changelog should exist alongside the instance), surface a one-line warning and proceed with the rest of the view. Don't halt.

### Step 7: Surface the full view

```
# {name} (`{slug}`)

Template: {template_slug} v{template_version}
Created: {created_date} by {created_by_display_name}
Status: {active | archived}

## Template fields

| Field | Value |
|---|---|
| company_name | Acme Pharma Inc. |
| primary_contact_name | Jane Smith |
| ... | ... |

## Extension fields

| Field | Value |
|---|---|
| office_location | Boston |
| ... | ... |

## Recent activity ({changelog_tail} entries)

- 2026-05-14 (Bill): field_edited — primary_contact_email changed
- 2026-05-13 (Bill): created
- ...
```

End with next-action suggestions:

> *"Edit this client: `@ai:edit-client {slug}`. Manage access: `@ai:view-permissions {slug}` / `@ai:grant-permission {slug}`. Soft- or hard-delete: `@ai:delete-client {slug}`."*

---

## Directives

### Behavior

Read-only and unobtrusive. The visibility-floor view is the most consequential UX moment in this task — it's where a member learns "this client exists but I don't have access." Be matter-of-fact, not apologetic; the visibility floor is by design.

If the caller is the creator of the client (matches `created_by` in the public-index entry) and somehow gets a permission-denied on the instance file, that's a data-integrity bug — surface it as: *"You're listed as this client's creator but your read failed. This shouldn't happen — please report it to your admin."*

### State Management

Not stateful. Each invocation reads fresh.

### Constraints

- **Never write anything.** Read-only by definition.
- **Never call permission-modifying ops.** This task does not grant, revoke, or change ACLs.
- **Never expose other instances' data** as part of this task — `view-client` is one instance at a time.
- **Never bypass the visibility floor** — if the caller lacks view perm on the instance, do not attempt to read its data through any side channel.

### Edge Cases

- **Slug doesn't exist.** Step 3 catches it.
- **Public-index entry exists but the instance folder is missing.** Step 5 surfaces this as an error; recommend the caller contact an admin.
- **Caller has view perm but the changelog read fails.** Step 6 warns and proceeds with current state only.
- **Multiple clients match the resolved name.** Step 2 disambiguates by asking for the slug. Names are not unique (the duplicate-name check at creation is informational, not blocking).
- **Caller is the creator but lacks view perm.** Data-integrity warning per the Behavior section.
- **Status is `archived`.** Render the same view but with the archived marker prominently displayed: *"This client is archived. To unarchive, run `@ai:edit-client {slug}` and reset status to active."*
