---
name: client-intelligence-tutorial
version: 1.0.0
type: skill
collection: client-intelligence
description: Explains the client-intelligence collection to members — its concepts, workflows, and how clients, templates, permissions, and the visibility floor work together — through a guided tour or targeted answers to specific questions.
stateful: false
always_on_eligible: false
dependencies:
  skills: []
  tasks: []
external_dependencies: []
---

## About This Skill

The client-intelligence collection is your org's primary repository of client-level knowledge. Members create, edit, view, and manage records of the clients the organization works with. The collection sits on top of native filesystem permissions — Drive ACLs enforce who can see and edit which clients — and exposes a "visibility floor" where client names are universally discoverable to all collection members while data is gated per-instance.

This skill explains the collection — its concepts, the tasks it offers, and how the pieces fit together. It does not perform operations; for those, use the specific task aliases like `@ai:create-client` or `@ai:list-templates`.

### When This Skill Is Active

When invoked, Claude shifts into explanatory mode. The skill remains active for the tutorial conversation.

### What This Skill Does Not Cover

This skill covers concepts and workflows specific to client-intelligence. It does not cover the broader access-control project or the underlying Drive permission model beyond what members need to use this collection. It does not cover internal file formats or the implementation details of the permission-change-helper. It does not cover troubleshooting filesystem connectivity.

---

## Directives

### Behavior

When invoked, determine whether the member wants a guided tour or has a specific question.

For a guided tour: run the structured tour sequence below. Check in after each topic — *"Should I continue, jump to a specific topic, or wrap up?"*

For a specific question: answer directly, drawing from the same material below. Don't read out the whole tour.

If the member is an admin, additional topics about template authoring and admin lifecycle become relevant; adapt accordingly. Admin status is "do they have Drive Editor on `/shared/client-intelligence/templates/`" — you can confirm via `aifs_get_permissions` if asked.

Read `/shared/client-intelligence/collection-state.json` and `/shared/client-intelligence/config/default-permissions.json` before responding when concrete examples matter, so what you describe reflects the actual install state (number of admins, default policy preset, whether the example template is installed).

### Guided Tour Sequence

Seven topics in order. After each, check in.

**Topic 1: What the client-intelligence collection does**

The collection gives your org a structured way to manage client records. Members create one entry per client — a template-defined record with fields you choose — and then edit, view, and (eventually) archive or delete those records over time. Each client lives in its own folder on the org's shared filesystem; Drive ACLs control who can see which clients.

The collection has three primary concepts: **templates** (admin-authored definitions of what fields a client carries), **client instances** (specific clients created from a template, optionally extended with member-added fields), and **per-instance permissions** (who can view, edit, and delete each client). Members create clients; admins manage templates and the default permission policy.

If the member is new, mention that the example template `example-client` ships at install time so the collection is exercisable immediately.

**Topic 2: Templates and what they define**

Templates are blueprints for clients. An admin authors a template (`@ai:create-template`) and specifies which fields a client created from this template will carry. Each field is either **mandatory** (must be filled in at creation) or **optional** (may be skipped). Templates have version history — every edit produces a new version with a snapshot retained.

When a member creates a client (`@ai:create-client`), they pick a template, fill in the fields, and optionally add their own **extension fields** beyond what the template defines. Extension fields are how members add per-client information that isn't part of the structured template (a specific business context, a free-form note, anything not anticipated by the template author).

Admins can edit templates (`@ai:edit-template`) and at each edit choose whether to migrate existing client records to the new schema (Migrate) or leave them on the previous version (No-impact). Migrate is heavier — it touches every existing client created from that template — and offers a dry-run preview before any write.

To see the available templates: `@ai:list-templates`. To inspect a specific one: `@ai:view-template {slug}`.

**Topic 3: Creating, viewing, and editing clients**

To add a new client: `@ai:create-client`. You'll pick a template, fill in the template fields, optionally add extension fields, and give the client a name. The name is what other members will see in lists (more on that in Topic 5). You can also optionally grant other members access at creation time — they'll be added to the new client's ACL automatically.

To see a client: `@ai:view-client {slug}`. You get the full data if you have view permission on that client; otherwise you see only the name (the visibility floor — Topic 5).

To enumerate all clients: `@ai:list-clients`. Clients you can view show full row; clients you can't show name-only.

To modify a client: `@ai:edit-client {slug}`. You can change template field values, add extension fields, and remove extension fields. You cannot structurally modify template-defined fields (add new template fields, rename, change mandatory/optional) — those changes belong on the template via `@ai:edit-template`.

**Topic 4: Per-instance permissions and the permission-change-helper**

Each client instance has its own Drive ACL. The three permissions in the client-intelligence model are **view**, **edit**, and **delete**. When a client is created, the creator unconditionally gets all three. The collection-level default permission policy (configured at install, editable via `@ai:edit-default-permissions`) layers additional grants on top. And the creator can include explicit per-creation grants in the create flow.

To grant access to someone else after creation: `@ai:grant-permission {slug} {member}`. To revoke: `@ai:revoke-permission {slug} {member}`. To see who currently has access: `@ai:view-permissions {slug}`.

A safety note about permission changes: granting and revoking are **never done silently by Claude**. The agent prepares the proposed change and surfaces a clickable link in chat; you click it to open a review page in your browser, then click Accept to apply the change with your own credentials. This is by design — agents are categorically prohibited from making security-changing calls on the user's behalf. The flow takes a few seconds longer but keeps the security boundary intact.

**Topic 5: The visibility floor**

This is the part of the collection people find most surprising on first encounter. **Every collection member can see every client's name**, regardless of whether they have view permission on that client. They just can't see the data. The name is visible; the contents are gated.

Why this design: it enables duplicate-name detection across the entire collection (if two people independently try to add the same client, the system warns them), it makes the collection's universe of clients discoverable, and it forces the operational question of confidentiality to a separate layer.

For sensitive engagements where even the client name is confidential, the convention is to use a **codename**. *"Project Falcon"* instead of *"Acme M&A advisory."* The system has no special confidentiality mechanism beyond this; codenames are the agency's operational choice, not a platform feature.

(Note: in V1, the visibility floor on **names** is fully enforced via separate ACLs on the public-index file. The visibility floor on **data** is degraded until access-control Phase 5 ships the helper-spec extension for `inherit: false` — currently, every collection member has effective Drive Writer on every client instance via inheritance from the parent folder. The codename pattern is the only data-confidentiality mechanism that works pre-Phase-5. This limitation is tracked under the `builder-profile-adaptive-dev-process` umbrella in core-improvements.)

**Topic 6: Deletion — soft and hard**

To delete a client: `@ai:delete-client {slug}`. You'll choose a mode:

- **Soft delete** flips the client's status to archived. The data and changelog are preserved on the filesystem; the client just stops appearing in default lists. There is no V1 unarchive task — admins can manually flip the status field if needed (a future version may add an unarchive task).
- **Hard delete** is permanent. The instance folder and all data are removed. A finalizing entry is appended to the collection-wide template changelog as the surviving audit record. There is no recovery path. Hard delete requires explicit literal-phrase confirmation: typing *"yes, permanently delete {slug}"* exactly.

Both modes require Delete permission on the instance (which V1 maps to Drive Writer).

**Topic 7: Admin tasks**

If you're a collection admin (Drive Editor on `templates/` and `config/`), you have additional capabilities:

- `@ai:create-template`, `@ai:edit-template` — author and modify templates.
- `@ai:edit-default-permissions` — modify the policy applied to every newly created client.
- `@ai:add-admin {member}`, `@ai:remove-admin {member}` — manage who else has admin role. Both go through the permission-change-helper (browser click). A last-admin guardrail prevents removing the only admin.

Admin role is derived from the filesystem — you're an admin if you have Drive Editor on the right folders. There's no separate admin list to maintain. Adding an admin grants those folder permissions; removing them revokes.

After Topic 7, ask whether the member wants a recap of any topic, has a specific scenario to walk through, or is ready to start using the collection.

### Question-Answering Mode

If the member asks a specific question instead of requesting a tour, answer directly:

- **"How do I add a client?"** → Topic 3, focus on `create-client`.
- **"Who can see my clients?"** → Topic 4 + Topic 5 together.
- **"What's a template?"** → Topic 2.
- **"How do I get rid of a client?"** → Topic 6.
- **"Why do I have to click a link to share a client?"** → Topic 4, permission-helper paragraph.
- **"I see a client name I don't recognize"** → Topic 5, visibility floor explanation.
- **"How do I become an admin?"** → Topic 7, mention that an existing admin runs `@ai:add-admin`.

For questions that span multiple topics, give the integrated answer rather than reading topics in order.

### Constraints

- **Never perform operations from inside this skill.** This is a tutorial — point at the right task alias, don't run it.
- **Never expose implementation details** (file paths, JSON schemas, internal changelog format) unless the member specifically asks about implementation.
- **Always disclose the V1 inheritance limitation** when explaining the visibility floor or permissions, so members don't have a false sense of data-confidentiality.
- **Always acknowledge the permission-helper friction** in Topic 4. The browser-click step is by design; explaining it heads off confusion when members hit it.

### Edge Cases

- **Member asks about V2 / future features.** Defer: *"That's not in V1. The collection's ROADMAP file lists what's coming."*
- **Member is confused about admin vs. member.** Clarify by example — admins author templates and set policy; members create clients and use templates. Both can share their own clients with each other; neither can see another's clients without an explicit grant.
- **Member asks why agents can't directly make permission changes.** Reference Topic 4's safety-boundary paragraph; expand with the prompt-injection rationale only if they push for more.
- **Member is exploring without a clear question.** Offer the guided tour starting at Topic 1.
tion.** Offer the guided tour starting at Topic 1.
