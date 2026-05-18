---
name: delete-client-setup
type: setup
version: 1.0.0
collection: client-intelligence
description: Setup for the delete-client task — no member-specific parameters. Validates that the collection is installed and view-client is available.
target: delete-client
target_type: task
upgrade_compatible: true
---

## Setup Overview

Delete Client requires no member-specific configuration. The task interviews the member for the deletion mode and confirmation at invocation time. This setup verifies the collection is installed, confirms the `view-client` task dependency, registers the alias, and confirms availability.

---

## Pre-Setup Checks

- Collection setup has been completed: verify `/shared/client-intelligence/collection-state.json` exists via `aifs_exists`. If not: *"The client-intelligence collection is not yet installed on your org. An org admin must install it before clients can be deleted."*
- Remote filesystem is accessible: test with `aifs_auth_status()`. If not: *"Please check your remote filesystem connection or run `@ai:member-bootstrap`."*
- The `view-client` task is installed. If not, instruct: *"Delete Client depends on View Client. Reinstall the collection so all V1 tasks are present."*

---

## Parameters

No member-defined parameters.

---

## Setup Completion

1. Validate remote filesystem access.
2. Confirm `/shared/client-intelligence/collection-state.json` exists.
3. Confirm `view-client` is installed.
4. Register entry in `member-index.json` with alias `@ai:delete-client`.
5. Confirm to member: *"Delete Client is ready. Say `@ai:delete-client {slug}` to remove a client. You'll choose soft (archive) or hard (permanent) at invocation. Hard delete requires explicit literal-phrase confirmation."*

---

## Upgrade Behavior

### Preserved Responses

No member-specific responses to preserve.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: migration notes will be added here as new versions of the task are published. A planned post-V1 addition is an `unarchive-client` task; once that ships, soft-deleted clients will have a recovery path via task, not just admin intervention.
