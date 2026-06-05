---
name: client-intelligence-tutorial
type: skill
version: 2.0.0
collection: client-intelligence
description: Explains the client-intelligence collection to members — templates, the two visibility tiers (private in your own Drive vs. org-public in the commons), the universal visibility floor, sharing, custody, and lifecycle — through a guided tour or targeted answers.
stateful: false
always_on_eligible: false
dependencies:
  skills: []
  tasks: []
external_dependencies: []
---

## About This Skill

The client-intelligence collection is your org's repository of client-level knowledge. Members create one record per client from admin-authored templates, then view, edit, share, and manage those records over time. Since 2.0, access works on a **two-tier model**: a client is either *private* (it lives in your own My Drive — yours until you share it) or *org-public* (it lives in the org commons — everyone's, uniformly). Either way, its NAME is always discoverable org-wide: the **universal visibility floor**.

When giving concrete examples, read `/shared/client-intelligence/collection-state.json` and the pointer index (`public-index/instances/`) so examples reflect the actual install.

## Topics

**Topic 1: What this collection is for**

One structured record per client relationship: template-defined fields (admins decide what a client record carries), plus your own extension fields. Create, view, edit, share, archive. The org-wide directory (`@ai:list-clients`) shows every client's name and owner — so nobody creates a duplicate record and everybody knows who owns which relationship.

**Topic 2: Templates (admins)**

Admins author templates (`@ai:create-template`) defining mandatory and optional fields. Instances bind to the template version they were created from. Unchanged from 1.x.

**Topic 3: The two tiers — where a client lives**

- **Private (the default).** Created in YOUR own My Drive (`Agent-Index-Private/clients/`). You are the Drive owner: nobody can see the data until you share it, and you alone control access. Use for prospecting, sensitive negotiations, personal pipeline.
- **Org-public.** Created in the org commons (`/shared/client-intelligence/instances/`). Every member can view AND edit it (uniformly — there is no per-person gating in the commons, by design). Every edit is attributed in the client's changelog. Use for established, revenue-bearing org relationships.

**Custody is the real difference.** An org-public client belongs to the org — it survives anyone's departure. A private client belongs to YOU: if you leave the org, the data and any shares you made go with your account, outside org control. Rule of thumb: if the org would need it after you're gone, make it org-public. You can move a client between tiers anytime: `@ai:transition-client {slug}` (note: taking a client private doesn't un-publish what the org already saw).

**Topic 4: Sharing a private client**

Same vocabulary as everywhere in agent-index: *share with X* → X can read; *make X a collaborator* → X can read and write. `@ai:grant-permission {slug}` (owner-only — it's your folder; you click Accept on the permission review page and the grant applies with your credentials). `@ai:revoke-permission` is the inverse. `@ai:view-permissions` shows who has what, cross-checked against live Drive state.

**Topic 5: The universal visibility floor**

Every client — both tiers, including fully private ones — has a pointer record in the org-wide directory: name, owner, template, status, tier. That's the floor: anyone can discover that the relationship exists and who owns it; the DATA is gated by tier. Two consequences: (1) duplicate detection works — create-client warns you if the name already exists; (2) use a **codename** if even the client's name is confidential. Unlike 1.x there is no ACL fine print here — private data is structurally invisible (it's in the owner's Drive), and the old "ancestor leak" caveat is gone.

**Topic 6: Lifecycle**

Archive (default "delete") hides a client from default listings; data stays; unarchive anytime. True deletion exists only where someone really holds delete power: owners may permanently delete their own private clients (the directory keeps a name-only record marked deleted); the commons is archive-only for members. The floor record never disappears — the org's history of "we had this relationship" is permanent.

**Topic 7: Member lifecycle**

When a member with private clients is offboarded, their shares keep working (Drive permissions on their content) but the org loses governance over them — the directory annotates those records "owner departed," and any current recipient can adopt one by copying it into their own space and re-sharing. Org-public clients are unaffected by departures.

**Topic 8: Admin tasks**

- `@ai:create-template`, `@ai:edit-template` — author and modify templates.
- `@ai:edit-default-permissions` — set the org's default for the creation-time visibility prompt (private-first by default). It no longer writes any ACL policy.
- `@ai:add-admin` / `@ai:remove-admin` — manage the collection-admin roster.

## Q&A routing

- "I see a name I don't recognize" → Topic 5 (the floor).
- "Why can't I gate who edits this?" → it's org-public; Topic 3 + transition-client.
- "Can I see X's client?" → ask X; granting is owner-run (Topic 4).
- "What happens if I leave?" → Topic 7 / custody (Topic 3).
- "How do I delete a client?" → Topic 6.
