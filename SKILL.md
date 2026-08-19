---
name: list-builder
description: >
  Build, validate, and maintain Git-backed Gabriel Operator data lists by
  editing portable assets/list.json definitions. Use this skill when defining
  a list schema or a portable Pipeline/List binding. Runtime rows remain in the
  target environment and are never included in portable definition repos.
metadata:
  author: gabriel-operator
  version: "1.0"
  compatibility: Requires Node.js 16+ for validation scripts.
---

# List Builder

## Portable Git contract (schema v2)

```json
{
  "schemaVersion": 2,
  "resourceKey": "list.example.cases-v2",
  "runtimeDataPolicy": "definitions_only",
  "list": {
    "name": "Cases",
    "columns": [],
    "pipelineRef": { "kind": "pipeline", "resourceKey": "pipeline.example.cases-v2" }
  }
}
```

- Never commit `listId`, `pipelineId`, `collectionId`, `pageId`, `userId`, record IDs, run IDs, or runtime snapshots.
- Use `pipelineRef` for the reciprocal Pipeline dependency. The Pipeline uses `storage.listRef` with this List's resource key.
- Keep column keys and option values stable; these are logical schema identifiers.
- Store rows, runs, evidence, media, and audits only in the environment-local database. Do not create or sync `data/records.json` for portable repositories.
- When publishing through a Persona bundle, pin the List repository's exact commit `revision` and the SHA-256 `definitionFingerprint` of `assets/list.json` in the Persona registry.
- Require `runtimeDataPolicy: "definitions_only"` and run portability validation before publishing.
- Legacy schema v1 data import remains same-environment compatibility only. Never use it for cross-environment assets or new exports.

## Using this skill in coding agents

Gabriel Operator skills are designed for Claude Code, Codex, Cursor, Hermes, OpenClaw, and any agent that supports skill packs. Work in the git-backed list repository connected to your Gabriel list.

### Install the skill pack

| Agent | Install |
|-------|---------|
| **Claude Code** | `npx skills add go-code-bot/list-builder` |
| **Codex** | `codex plugin marketplace add Gabriel-Operator/gabriel-operator-coding-agent-plugin --sparse .agents/plugins` then install the Gabriel Operator plugin |
| **Cursor** | `npx github:go-code-bot/list-builder add ./my-list` or copy into `.cursor/skills/list-builder/` |
| **Hermes / generic CLI** | `npx github:go-code-bot/list-builder add ./my-list` |
| **OpenClaw** | `npx skills add go-code-bot/list-builder` then `openclaw gateway connect --url https://your-openclaw-gateway` |
| **Gabriel Operator monorepo** | `cp -R server/skills/list-builder ./your-git-repo/` |

Alternative curl installer:

```bash
curl -fsSL https://raw.githubusercontent.com/go-code-bot/list-builder/main/install.sh | bash
```

### Modify with your coding agent

1. Open the git-backed list repository.
2. Tell your agent: *"Read `SKILL.md` and update the portable `assets/list.json` definition for \<describe the list change\>. Do not export runtime rows."*
3. Validate before committing:
   ```bash
   node scripts/validate-list.js assets/list.json
   npm run audit:portable-assets -- assets/list.json
   ```
4. Commit and push to the default branch.

**Example prompts:**
- *"Add a select column for portion size with options small, medium, large."*
- *"Generate ten grocery rows with realistic item names and quantities."*
- **OpenClaw:** *"Update the portable assets/list.json definition, run the definition and portability validators, and prepare it for Git sync."*

### Sync to Gabriel

1. Run the definition and portability validators.
2. Commit and push to the default branch.
3. Import or sync the definition. Runtime rows remain in that environment.

## Git-backed list repositories

When this skill is materialized as a Git repository for one list, the repo
contains the scaffold plus `assets/list.json`. Each list is **personal to the
user that created or imported it** — the same list payload can be imported by
another user, and each user gets their own independent Git binding. The
runtime reads the synced default branch / database projection; non-default
branches are for authoring and review.

Use this skill when editing `assets/list.json` for a Git-backed list.

### Inside a Persona workspace

This repository is usually a **git submodule** of an AI Persona repository, at
`references/lists/<resource-key>/`. The parent owns `references/registry.json`, which pins
this repo's commit `revision` and the SHA-256 `definitionFingerprint` of
`assets/list.json`. After changing the definition, commit and push here **first**, then
publish the parent workspace so gitlinks and fingerprints advance together. Until you do,
the Persona still resolves the previous commit. The parent workspace is coordinated
authoring, not an atomic multi-repo commit.

## Mental Model

- One repository owns one list.
- A list is the **column schema + display name + optional pipeline binding** for
  rows that live in an `app_data_records` collection. The list does not own
  rows — it owns the *shape* and the *identity* of the column set.
- `list.pipelineRef` declares a portable Pipeline relationship. Import resolves
  the reciprocal references to one local collection in the target environment.
- Runtime table rows cannot be authored in a portable definition repository.
- Per-user UI state still does not belong in Git.
- Keep `resourceKey` stable. Change it only when intentionally forking a model.

## Canonical File

For schema edits, edit:

```text
assets/list.json
```

Expected wrapper:

```json
{
  "schemaVersion": 2,
  "resourceKey": "list.grocery.items-v2",
  "runtimeDataPolicy": "definitions_only",
  "list": {
    "name": "Grocery List",
    "description": "Personal grocery items",
    "columns": [],
    "pipelineRef": { "kind": "pipeline", "resourceKey": "pipeline.grocery.automation-v2" }
  },
  "commitMessage": "Update list definition"
}
```

## Columns

Columns are the table schema — same shape as pipeline columns so the two
worlds are interchangeable.

```json
{
  "key": "item_key",
  "label": "Item Key",
  "type": "text",
  "required": true,
  "options": ["small", "medium", "large"]
}
```

Allowed types: `text`, `number`, `boolean`, `date`, `select`. The `options`
array is only meaningful for `select`-typed columns.

## Common Edits

Rename the list:

1. Update `list.name`. Optionally tweak `list.description`.
2. Leave the top-level `resourceKey` unchanged.

Add a column (standalone list):

1. Append a new entry to `list.columns[]`.
2. Use a unique `key`.
3. Pick the simplest matching `type`.

Bind / unbind a pipeline:

1. Set `list.pipelineRef` to the Pipeline resource key you want to inherit columns from.
   Existing `list.columns[]` becomes the cached snapshot — at runtime, the
   pipeline's columns shadow it.
2. To detach, clear `list.pipelineRef` and freeze the columns you want to keep
   into `list.columns[]`.

Model existing-case lineage:

1. Mirror every field referenced by a pipeline `existingCasePolicy` in the
   bound List schema. Identity, attempt, mode, source-record, and source-run
   fields are text columns; reused answer keys are a native JSON column.
2. Keep previous cases immutable. A reuse choice creates a new row whose
   lineage columns reference the source row and run; it never reopens or
   overwrites the source row.
3. Do not store approval grants, submission keys, receipts, evidence, errors,
   or pipeline state in launch-decision metadata. Those remain governed fields
   on the new case row.
4. Never add runtime case rows to Git while modelling this schema.
5. In the UI, define the policy from **Pipeline → Manage → Config → Existing-case
   detection**. Every selectable field comes from the Pipeline/List column schema;
   add or correct the shared column first instead of typing an undeclared key.
6. `locatorField` is a resource locator, not a universal `form_url` constant. Lists
   for other domains may expose `product_url`, `listing_url`, or another stable URL,
   with a separate text identity column for the canonical hash.

## Validation

Run:

```bash
node scripts/validate-list.js assets/list.json
```

The validator rejects missing required fields, duplicate column keys, invalid
column types, and malformed schema wrappers.

## Runtime rows

Do not create, edit, validate, or commit `data/records.json` in a portable List repository.
Create or mutate rows only through the environment's authenticated List/Pipeline runtime.
