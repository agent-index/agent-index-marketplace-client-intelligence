---
name: client-brief
type: task
version: 1.0.1
collection: client-intelligence
description: Produce a shareable client brief document (markdown or docx) from a client instance — template fields, extensions, recent activity — brand-styled when the org has a brand-book provider registered.
stateful: false
produces_artifacts: true
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies: []
reads_from: "/shared/client-intelligence/, instance locations (tier-resolved)"
writes_to: null
---

## About This Task

Renders a client instance into a polished, shareable DOCUMENT — the collection's first file artifact and the org's first brand-book consumer surface. Access is tier-mechanical exactly as view-client; the document lands in the member's LOCAL workspace (sharing it is the member's choice).

### Inputs
Client (slug or name); format (`markdown` | `docx`, default markdown); optional sections filter.

### Outputs
- `members/{member_hash}/client-intelligence/briefs/{slug}-brief-{date}.{ext}` (LOCAL, native file tools)

## Workflow

### Step 1: Pre-flight
`aifs_auth_status`; collection installed check.

### Step 2: Resolve and Read the Instance
Identical mechanics to view-client Steps 2–3 (pointer → tier-resolved read of `instance.json` + `changelog.json`; private-tier ACCESS_DENIED → floor view guidance and HALT — no document for instances you can't read).

### Step 3: Resolve the Brand (inline — int4, 2.1.1)

Resolution is inlined here rather than referencing `/internal/resolve-brand.md`, because members cannot read collection `/internal/` files at runtime (they aren't synced locally and can't be path-resolved on the org remote — bug 20260608 int4). The steps:

1. **Find the provider.** `aifs_read("/org-config.json")` → `capability_providers["brand-book"].providers`. Empty/absent → **no brand**: compose UNBRANDED with a one-line notice ("Brand styling not configured — install and register a brand-book provider to enable it"); skip the rest of Step 3. Exactly one provider → use it. Multiple → V1 isn't configured for multi-provider binding; compose unbranded with a notice.
2. **Resolve the provider's read base (id-anchored, core 3.10.1).** Find the provider collection's entry in `installed_collections[]`; `base = "id:{folder_id}"` if present, else `/{provider_collection}`. `brand_book_version` = the provider's `capability_version` from the registry (authoritative; do NOT depend on name-resolving the provider's collection.json — bug db13).
3. **Fetch once (read-only provider ops, executed from `{base}/api/...`):** `get-brand-guidelines(artifact_type:"client-brief", format, sections:all)`; `get-template("client-brief", format)` (optional — absent → native structure); for each element the composition uses, `get-element(name, format)` (`rendering_missing`/`found:false` → native default for that element).
4. **Personal-element precedence.** Read LOCAL `members/{member_hash}/brand-book/personal/elements/{slug}.json` (native tools): if registry `provider_config.brand_usage == "optional"`, personal overrides org; if `"required"`, org wins and personal applies only to element types the org hasn't defined.
5. **Slot values** come from THIS instance's `branding/` (read with the same tier mechanics as the instance: org-public path or `id:{folder_id}`): `client-logo.*` and `branding.json` `client_display_name`. A slot with no value: render a marked placeholder if the template marks it required, else omit.

**Failure rule:** any failure in steps 1–5 (unreadable registry, corrupt provider files, missing op) degrades to UNBRANDED composition with a specific notice — brand resolution never blocks or fails the brief.

### Step 4: Compose
Sections (template-driven when get-template found `client-brief`, else native order): title block (client display name; `client-logo` slot per template placement), summary (template fields), extensions, status & ownership (metadata-block element), recent activity (table element, last 10 changelog entries), footer. Apply org voice to connective prose. For docx: use the document-generation skill per its SKILL.md after composition is fully determined.

### Step 5: Write + Report
Write locally (native tools). Report the path, the branding stamp (or "unbranded — {reason}"), and the one-line branded-vs-native summary from the resolve pattern.

## Directives

### Behavior
The brief is a snapshot for humans — favor readable prose over raw field dumps; include the as-of date prominently.

### Constraints
LOCAL output only; never write to shared spaces or the instance. Never include data the caller's tier couldn't read interactively. Brand failures degrade, never block (resolve-brand failure rule).

### Edge Cases
Instance with no `branding/` → omit slots per template rules (notice only if the template marks a slot required). Unknown format → name the two supported. Empty changelog → omit the activity section.
