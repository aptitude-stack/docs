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
- The folder does not contain `README.md`.
- `SKILL.md` starts with YAML frontmatter delimited by `---`.
- Frontmatter includes `name`, `description`, and a `metadata` block.
- `name` matches the folder name and is kebab-case.
- `description` explains what the skill does and when to use it.
- `metadata.tags`, `metadata.inputs_schema`, and `metadata.outputs_schema` are
  present.
- The body after frontmatter contains actual instructions.
- Optional files live under `scripts/`, `references/`, or `assets/`.
- Generated `.publisher_artifacts/` output is not treated as skill content and
  is excluded from the uploaded bundle.

## Folder Layout

Recommended layout:

```text
my-skill/
  SKILL.md
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
| `scripts/` | Optional executable or helper scripts. |
| `references/` | Optional markdown or source material the skill can reference. |
| `assets/` | Optional binary or visual assets. |
| Other `.md` files | Companion markdown included in word/token metrics. |
| Other files | Included in inventory and uploaded bundle unless under `.publisher_artifacts/`. |

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
bad: python.review
```

The frontmatter `name` must exactly match the folder name.

## SKILL.md Structure

`SKILL.md` must start with frontmatter and then contain markdown instructions:

```markdown
---
name: python-review
description: Helps review Python services when users ask for code quality, test, or architecture feedback.
metadata:
  tags: [python, review, testing]
  inputs_schema: {"type": "object", "properties": {"request": {"type": "string"}}}
  outputs_schema: {"type": "object", "properties": {"findings": {"type": "array"}}}
  maturity_score: 0.8
  security_score: 0.9
  token_estimate: 1200
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

## Frontmatter Fields

### Required Top-Level Fields

| Field | Required | Rule |
| --- | --- | --- |
| `name` | Yes | Non-empty kebab-case string matching the folder name. |
| `description` | Yes | Non-empty string under 1024 characters. |
| `metadata` | Yes | Mapping containing publish metadata. |

### Required Metadata Fields

| Field | Required | Rule |
| --- | --- | --- |
| `metadata.tags` | Yes | Non-empty list of strings. |
| `metadata.inputs_schema` | Yes | JSON-object-compatible mapping. |
| `metadata.outputs_schema` | Yes | JSON-object-compatible mapping. |

### Optional Metadata Fields

| Field | Required | Rule |
| --- | --- | --- |
| `metadata.token_estimate` | No | Integer. Recorded as declared token estimate; publisher still computes its own estimate. |
| `metadata.maturity_score` | No | Number from `0.0` to `1.0`. |
| `metadata.security_score` | No | Number from `0.0` to `1.0`. |
| `compatibility` | No | String from 1 to 500 characters if present. |
| `license` | No | String; copied into publisher metadata extras. |

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

`inputs_schema` and `outputs_schema` should be JSON-schema-like objects. The
current publisher only requires that they parse as mappings, but authors should
make them useful enough for reviewers, registry users, and downstream tools.

Minimal example:

```yaml
metadata:
  inputs_schema: {"type": "object", "properties": {"request": {"type": "string"}}}
  outputs_schema: {"type": "object", "properties": {"result": {"type": "string"}}}
```

Prefer explicit properties over empty objects when the skill expects a concrete
workflow.

## Files Included In The Bundle

The uploaded bundle is built from the skill folder as deterministic
`skill.tar.zst` bytes.

The archive:

- includes files from the skill folder,
- excludes `.publisher_artifacts/`,
- stores files under the archive root `skill-bundle/`,
- normalizes file metadata for deterministic output.

Generated publisher artifacts are local audit records only. They are not part
of the uploaded skill.

## Publisher Artifacts

During `inspect` or `publish`, the publisher writes `.publisher_artifacts/`
inside the skill folder.

Common artifacts:

| Artifact | Purpose |
| --- | --- |
| `00_inventory.json` | File inventory, git provenance, and parsed package context. |
| `01_identity.json` | Slug, version, and publish intent. |
| `02_metadata.json` | Extracted metadata and publisher-computed token/word metrics. |
| `04_security.json` | garak security result. |
| `05_validation.json` | Validation errors, warnings, checks, and notes. |
| `05_performance_exam.json` | Optional Upskill metrics. |
| `03_ranking.json` | Weighted quality score and publish decision. |
| `07_compression.json` | Local delivery-package compression stats. |
| `07_delivery_package.zst` | Compressed delivery payload JSON for local traceability. |

These files are useful for review and debugging. They should not be committed
as part of the skill content unless a separate review workflow explicitly asks
for them.

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

Relationships are not accepted from `SKILL.md`, CLI flags, or any current
publisher parser path. The publisher currently sends the relationship block as
empty generated defaults:

```json
{
  "depends_on": [],
  "extends": [],
  "conflicts_with": [],
  "overlaps_with": []
}
```

Do not document dependencies or related skills in frontmatter expecting the
publisher to publish them. Relationship extraction needs a dedicated publisher
implementation before these values can come from skill author input.

## Common Failure Modes

| Failure | Fix |
| --- | --- |
| `SKILL.md must start with YAML frontmatter` | Put a `---` block at the top of `SKILL.md`. |
| `Frontmatter name should match the skill folder name` | Rename the folder or update `name`. |
| `Metadata is missing inputs_schema` | Add `metadata.inputs_schema`. |
| `Metadata is missing outputs_schema` | Add `metadata.outputs_schema`. |
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
