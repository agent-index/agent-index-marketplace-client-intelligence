---
name: client-brief-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup template for client-brief (no member-level parameters).
target: client-brief
target_type: task
upgrade_compatible: true
---

## Parameters

None — format is chosen per invocation; branding follows the org's brand-book registration.

## Setup Completion

1. Write the installed instance (using native file tools — LOCAL workspace) to `members/{member_hash}/skills/client-brief/`
2. Write `manifest.json`; register in `member-index.json` with alias `@ai:client-brief`.
3. Confirm: "Client Brief is ready — say 'client brief for {name}'."

## Upgrade Behavior

### Preserved Responses
None.
### Reset on Upgrade
Nothing.
### Requires Member Attention
Nothing.
### Migration Notes
None.
