# Resolve Brand (consumer boilerplate — copy-adapted from agent-index-core/templates/resolve-capability.md)

**Usage:** referenced by artifact-producing tasks in this collection: "Follow `/internal/resolve-brand.md` for format `{format}`, artifact type `{artifact_type}`." This is the reusable pattern for ANY collection consuming the `brand-book` capability — copy this file and adjust the collection name.

## Steps

### 1. Resolve the provider (single-provider auto-bind, V1)

`aifs_read("/org-config.json")` → `capability_providers["brand-book"].providers`.
- Absent / empty → **no brand**: proceed UNBRANDED with a one-line notice ("Brand styling not configured — install and register a brand-book provider to enable it."). This is `fallback: skip_with_notice` per our `requires[]` declaration. Done.
- Exactly one → use it: note `provider_collection` and `provider_config.brand_usage` (`required` | `optional`).
- Multiple → V1: surface "Multiple brand-book providers are registered; binding configuration isn't supported yet — using none." Proceed unbranded.

### 2. Fetch once per artifact (hold in session)

Read the provider's `collection.json` (`/{provider_collection}/collection.json`) → `provides[].operations` → implementing tasks. Execute per their definitions (they are read-only):
- `get-brand-guidelines(artifact_type, format, sections: all)` → guidelines + `brand_book_version`.
- `get-template(artifact_type, format)` → template + slots (optional op — absent/not-found → compose with this collection's native structure).
- For each element the composition needs: `get-element(name, format)`; `rendering_missing` or `found:false` → native default for that element.

### 3. Personal-element precedence

Read LOCAL `members/{member_hash}/brand-book/personal/elements/{slug}.json` (native file tools) for any element the composition uses:
- `brand_usage = optional`: personal definition WINS over the org element.
- `brand_usage = required`: org element wins; personal applies only where the org has no definition.

### 4. Slot values

Slot values come from THIS collection's context — for client-intelligence: the instance's `branding/branding.json` (`client_display_name`) and `branding/client-logo.*`, read with the SAME tier mechanics as the instance itself (org-public path or `id:{folder_id}` anchor). A slot with no value available: if `required: true` in the template → render a clearly-marked placeholder + note; else omit per the template's placement rules.

### 5. Compose, stamp, note

Apply voice to prose, tokens/elements per format, template structure when found. Stamp artifact metadata: `branding: {brand_book_version, brand_usage, composed: <date>}`. Report at the end which parts were branded vs native (one compact line).

**Failure rule:** ANY failure in steps 1–4 (unreadable registry, corrupt provider files, missing ops) degrades to unbranded composition with a specific notice. Brand resolution must never block or fail the artifact.
