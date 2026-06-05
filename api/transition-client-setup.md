---
name: transition-client-setup
type: setup
version: 2.0.0
collection: client-intelligence
description: Setup for the transition-client task
target: transition-client
target_type: task
upgrade_compatible: true
---

## Setup Overview
Validates that the collection is installed and the member's private space exists. No member-specific configuration.

## Pre-Setup Checks
- Collection installed (`/shared/client-intelligence/collection-state.json` exists)
- `member_folder_id` present in `member-index.json` (if missing, `@ai:update` self-provisions)

## Parameters
No member-configurable parameters.

## Setup Completion
1. Register entry in `member-index.json` with alias `@ai:transition-client`
2. Confirm to member: "You can move clients between private and org-public tiers with '@ai:transition-client'."

## Upgrade Behavior

### Preserved Responses
N/A.

### Reset on Upgrade
N/A.

### Requires Member Attention
None.

### Migration Notes
- New in collection 2.0.0.
