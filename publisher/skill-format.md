# Publisher Skill Format

This document defines the skill folder shape expected by `aptitude-publisher`
before a skill is considered ready for publish.

It is a publisher-side readiness guide. Passing these rules does not guarantee
registry acceptance: the registry still enforces token scope, namespace grants,
trust-tier rules, policy packs, payload validation, checksums, lifecycle state,
and audit.

## Ready For Publish Checklist

A publish-ready skill should satisfy all of these conditions:

- The skill lives in one folder whose name is kebab-case.
- The folder contains a primary `SKILL.md` file.
- The folder contains a required `aptitude.yaml` sidecar.
- The folder does not contain `README.md`.
- `SKILL.md` starts with YAML frontmatter delimited by `---`.
- Frontmatter contains the standard skill fields `name` and `description`; optional `license` and `compatibility` remain there too.
- `name` matches the folder name and is kebab-case.
- `description` explains what the skill does and when to use it.
- `aptitude.yaml` contains `version`, `intent`, `tags`, `inputs_schema`, and
  `outputs_schema`.
- The body after frontmatter contains actual instructions.
- Optional files live under `scripts/`, `references/`, or `assets/`.
- `agents/openai.yaml`, when present, remains an independent OpenAI configuration
  file.
- The legacy `.publisher_artifacts/` directory is preserved, ignored, and
  excluded from the uploaded bundle; the publisher does not read or write it.

## Folder Layout

Recommended layout:

```text
my-skill/
  SKILL.md
  aptitude.yaml
  agents/
    openai.yaml
  scripts/
    helper.py
  references/
    background.md
  assets/
    diagram.png
```

The publisher inventories the full folder:

| Path | Meaning |
| --- | --- |
| `SKILL.md` | Primary skill definition. Required. |
| `aptitude.yaml` | Aptitude publishing metadata. Required. |
| `agents/openai.yaml` | Independent OpenAI skill configuration, when present. |
| `scripts/` | Optional executable or helper scripts. |
| `references/` | Optional markdown or source material the skill can reference. |
| `assets/` | Optional binary or visual assets. |
| Other `.md` files | Companion markdown included in word/token metrics. |
| Other files | Included in inventory and uploaded bundle unless under the preserved `.publisher_artifacts/` directory. |

Evaluator inputs materialize file symlinks as independent copies. Directory symlinks are rejected with a diagnostic; use ordinary directories for evaluator input.

`README.md` is not allowed inside the skill folder. Put human-facing background
inside `SKILL.md` or `references/`.

## Folder Name

The skill folder name must match:

```text
[a-z0-9]+(-[a-z0-9]+)*
```

Examples:

```text
good: python-review
good: postman-primary-1774130709214
bad: Python Review
bad: python_review
bad: python-review
```

The frontmatter `name` must exactly match the folder name.

## SKILL.md Structure

`SKILL.md` must start with standard frontmatter and then contain markdown
instructions. Aptitude publishing metadata belongs in the adjacent
`aptitude.yaml`, not in this frontmatter:

```markdown
---
name: python-review
description: Helps review Python services when users ask for code quality, test, or architecture feedback.
compatibility: Works with Python repositories.
license: MIT
---

# Instructions

Use this skill when a user asks for Python code review, test review, or
architecture feedback.

## Process

1. Inspect the relevant files.
2. Identify correctness, security, and maintainability issues.
3. Return prioritized findings with file references.

## Example

User asks: "Review this FastAPI handler."

The skill inspects routing, request validation, dependency injection, error
handling, and tests.

## Troubleshooting

If no relevant files are available, ask for the target path before producing a
review.
```

The corresponding `aptitude.yaml` is:

```yaml
version: "0.1.0"
intent: create_skill
tags: [python, review, testing]
inputs_schema:
  type: object
  properties:
    request:
      type: string
outputs_schema:
  type: object
  properties:
    findings:
      type: array
token_estimate: 1200
maturity_score: 0.8
security_score: 0.9
```

## `SKILL.md` Frontmatter Fields

### Required Top-Level Fields

| Field | Required | Rule |
| --- | --- | --- |
| `name` | Yes | Non-empty kebab-case string matching the folder name. |
| `description` | Yes | Non-empty string under 1024 characters. |
| `metadata` | No | Other standard metadata entries remain independent of Aptitude publishing metadata. |

`name` and `description` are required. `license` and `compatibility` are
optional standard skill fields and stay in `SKILL.md`.

## `aptitude.yaml` Fields

`aptitude.yaml` is a flat YAML mapping beside `SKILL.md`. It is required even
when CLI or MCP values override some fields.

| Field | Required | Rule |
| --- | --- | --- |
| `version` | Yes unless supplied by `--version` | Non-empty string. |
| `intent` | Yes unless supplied by `--intent` | Non-empty string: `create_skill` or `publish_version`. |
| `tags` | Yes | Non-empty list of strings. |
| `inputs_schema` | Yes | JSON-object-compatible mapping. |
| `outputs_schema` | Yes | JSON-object-compatible mapping. |
| `relationships` | No | Mapping of relationship families; omitted families default to empty lists. |

Optional numeric hints are also accepted in the sidecar:

| Field | Required | Rule |
| --- | --- | --- |
| `token_estimate` | No | Integer authored hint; the publisher's computed or measured estimate takes precedence when available. |
| `maturity_score` | No | Number from `0.0` to `1.0`; computed scores take precedence when available. |
| `security_score` | No | Number from `0.0` to `1.0`; the authoritative security result takes precedence. |

The publisher rejects duplicate YAML keys, unknown sidecar fields, invalid
types, and malformed relationship selectors with file-specific errors. CLI and
MCP identity values override the sidecar; otherwise `version` and `intent` come
from `aptitude.yaml` and `slug` comes from the `SKILL.md` `name`.

`agents/openai.yaml` is independent and is not a source for Aptitude metadata.

## Description Requirements

The validation stage checks that `description` is useful trigger guidance, not
just a label.

It must:

- be non-empty,
- be shorter than 1024 characters,
- avoid `<` and `>` XML angle brackets,
- include an action marker such as `handles`, `creates`, `analyzes`,
  `manages`, `generates`, or `helps`,
- include a trigger marker such as `use when`, `when user`, `asks for`,
  `mentions`, or `says`.

Good:

```yaml
description: Helps review Python services when users ask for code quality, tests, or architecture feedback.
```

Weak:

```yaml
description: Python review skill.
```

## Reserved Names And Characters

The publisher blocks:

- `name` values containing `claude` or `anthropic`,
- frontmatter string fields containing `<` or `>`,
- folder names or `name` values with spaces, underscores, periods, or uppercase
  letters.

## Body Requirements

The body after frontmatter must not be empty.

The publisher warns when the body does not contain:

- an `# Instructions` heading,
- the word `example`,
- the word `troubleshooting`.

Warnings do not stop the validation gate, but they reduce the quality signal
used by the ranking stage. A publish-ready skill should include all three.

## Schema Fields

`inputs_schema` and `outputs_schema` in `aptitude.yaml` should be JSON-schema-like
objects. The current publisher only requires that they parse as mappings, but
authors should make them useful enough for reviewers, registry users, and
downstream tools.

Minimal example:

```yaml
inputs_schema:
  type: object
  properties:
    request:
      type: string
outputs_schema:
  type: object
  properties:
    result:
      type: string
```

Prefer explicit properties over empty objects when the skill expects a concrete
workflow.

## Relationship Fields

Relationships are optional in `aptitude.yaml`. Each family contains selectors
with a `slug` and optional `version` or `version_constraint`:

```yaml
relationships:
  depends_on:
    - slug: python-testing
      version: "0.1.2"
  extends: []
  conflicts_with: []
  overlaps_with: []
```

The publisher normalizes omitted families to empty lists and rejects malformed
selectors before external evaluation.

## Legacy Frontmatter Migration

The publisher does not fall back to Aptitude fields in `SKILL.md` frontmatter.
It rejects known legacy fields such as `version`, `intent`, `tags`, schemas,
relationships, and numeric hints, whether they appear at the top level or under
`metadata`. Unrelated standard `metadata` entries remain valid.

Move those fields manually to `aptitude.yaml`, leaving standard skill fields in
`SKILL.md`:

```diff
 ---
 name: python-review
 description: Helps review Python services when users ask for code quality, tests, or architecture feedback.
-metadata:
-  version: "0.1.0"
-  intent: create_skill
-  tags: [python, review]
-  inputs_schema: {}
-  outputs_schema: {}
-  maturity_score: 0.8
 compatibility: Works with Python repositories.
 license: MIT
 ---
```

Create `aptitude.yaml` beside it:

```yaml
version: "0.1.0"
intent: create_skill
tags: [python, review]
inputs_schema: {}
outputs_schema: {}
maturity_score: 0.8
```

The sidecar is included in the skill bundle and its checksum, so changing it
invalidates inspection reuse.

## Files Included In The Bundle

The uploaded bundle is built from the skill folder as deterministic
`skill.tar.zst` bytes.

The archive:

- includes `SKILL.md`, `aptitude.yaml`, `agents/openai.yaml` when present, and
  other files from the skill folder,
- excludes the preserved `.publisher_artifacts/` directory,
- stores files under the archive root `skill-bundle/`,
- normalizes file metadata for deterministic output.

The cache report and temporary evaluator workspaces are outside the skill
folder and are never part of the uploaded skill.

## Evaluation Report

Each canonical skill directory has one latest JSON report outside the source
tree. The cache root is the absolute `XDG_CACHE_HOME` when configured, or
`~/.cache` otherwise:

```text
<cache-root>/aptitude/publisher/<sha256(canonical-absolute-skill-directory)>.json
```

The report is atomically replaced and uses owner-only permissions. Its envelope
has `schema_version`, `skill_root`, `updated_at`, `status`, `stages`, `gates`,
`evidence`, `warnings`, `error`, and `inspection_receipt`. Status is one of
`running`, `ready`, `blocked`, or `failed`; exact evaluation decisions remain
inside the normalized evidence. When available, the signed inspection receipt
is nested under `inspection_receipt` and retains its MAC, expiry, configuration
fingerprint, identity, governance, and source-digest checks.

Raw evaluator transcripts, credentials, environment dumps, and temporary paths
are not retained. Evaluators run against temporary copies and working
directories outside the source tree; those files are cleaned after success,
failure, or timeout. Existing `.publisher_artifacts/` directories are preserved
for historical content, excluded from inventory and bundles, and never read or
written by the current publisher. There is no automatic migration or report
history.

## Validation Gates

The publisher runs these gates before a publish can proceed:

| Gate | Blocks when |
| --- | --- |
| Discovery | Skill root is invalid, `SKILL.md` is missing, frontmatter cannot be parsed, or inventory is incomplete. |
| Identity | Slug, version, or intent is missing. |
| Metadata | Name, description, tags, input schema, or output schema is missing; maturity/security score is out of range. |
| Security | garak did not run, did not score, produced invalid output, or returned `block`. |
| Validation | Validation produced any error. |

`review_required` from security or ranking is visible in output but is not
currently an upload stop in the publisher CLI. Registry governance may still
reject the request.

## Optional LLM Validation

The validation stage can also run token-backed semantic validation of
`SKILL.md`. When configured, LLM validation can add errors, warnings, and notes
to the same validation result. Treat this as an additional quality check on top
of the static folder/frontmatter/body rules above.

## Publish Request Shape

When `aptitude-publisher publish` uploads a skill, it sends:

```text
POST /skills/{slug}
Authorization: Bearer <publish-token>

metadata: application/json
bundle: application/zstd, filename skill.tar.zst
```

The metadata JSON contains:

```json
{
  "intent": "create_skill",
  "version": "1.0.0",
  "metadata": {
    "name": "python-review",
    "description": "Helps review Python services when users ask for code quality, tests, or architecture feedback.",
    "tags": ["python", "review", "testing"],
    "inputs_schema": {"type": "object"},
    "outputs_schema": {"type": "object"},
    "token_estimate": 1200,
    "maturity_score": 0.8,
    "security_score": 0.9
  },
  "governance": {
    "trust_tier": "untrusted",
    "namespace": "public",
    "artifact_origin": "internal",
    "policy_pack_slug": null,
    "provenance": {
      "repo_url": "https://github.com/example/repo.git",
      "commit_sha": "abc123",
      "tree_path": "skills/python-review",
      "publisher_identity": "ci"
    }
  },
  "relationships": {
    "depends_on": [],
    "extends": [],
    "conflicts_with": [],
    "overlaps_with": []
  }
}
```

Governance values come from CLI flags and git provenance. They are not inferred
from the skill body.

Relationships from `aptitude.yaml` are normalized into the registry payload.
When `relationships` is omitted, the publisher sends empty defaults for each
supported family:

```json
{
  "depends_on": [],
  "extends": [],
  "conflicts_with": [],
  "overlaps_with": []
}
```

Do not put Aptitude relationships in `SKILL.md` frontmatter. Legacy Aptitude
fields there are rejected and must be moved manually as shown above.

## Common Failure Modes

| Failure | Fix |
| --- | --- |
| `SKILL.md must start with YAML frontmatter` | Put a `---` block at the top of `SKILL.md`. |
| `Frontmatter name should match the skill folder name` | Rename the folder or update `name`. |
| `aptitude.yaml` is missing | Create the required sidecar beside `SKILL.md`. |
| `aptitude.yaml` is invalid | Use a flat mapping with the required fields and no unknown or duplicate keys. |
| `Metadata is missing inputs_schema` | Add `inputs_schema` to `aptitude.yaml`. |
| `Metadata is missing outputs_schema` | Add `outputs_schema` to `aptitude.yaml`. |
| `Legacy Aptitude field ... must be moved ...` | Move version, intent, tags, schemas, relationships, or numeric hints from frontmatter to `aptitude.yaml`. |
| `description must explain what the skill does and when to use it` | Add action and trigger language to `description`. |
| `README.md must not appear inside the skill folder` | Move docs into `SKILL.md` or `references/`. |
| garak did not produce a score | Configure garak target settings before publishing. |

## Authoring Recommendations

- Keep the skill focused and atomic.
- Put only operational instructions in `SKILL.md`; move long background content
  to `references/`.
- Keep scripts deterministic and document when the skill should call them.
- Keep schemas specific enough for downstream tooling.
- Include examples and troubleshooting so validation warnings do not become
  review churn.
- Run `aptitude-publisher inspect /path/to/skill` before publishing.
