---
name: client-brief
type: task
version: 1.0.0
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

### Step 3: Resolve the Brand
Follow `/internal/resolve-brand.md` with `artifact_type: "client-brief"`, the chosen format, and slot values from the instance's `branding/` (Step 4 there). The brief composes UNBRANDED with a notice when no provider is registered — never blocked.

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
