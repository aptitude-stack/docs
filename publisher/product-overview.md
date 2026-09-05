# Aptitude Publisher - Product Overview

The Aptitude Publisher is a local CLI for skill authors and CI pipelines. It
evaluates a skill folder, records a local audit trail, builds registry metadata,
creates a deterministic `skill.tar.zst` bundle, and uploads the result to an
Aptitude Registry.

It is not a server. Evaluation state is stored as one latest JSON report outside
the source tree, under the configured cache root.

## Users

| User | Goal |
| --- | --- |
| Skill author | Inspect a skill folder before publishing. |
| CI pipeline | Run the same checks and publish a versioned bundle. |
| Reviewer/operator | Inspect the latest evaluation report and registry payload inputs. |

## What It Owns

- Skill folder inventory, `SKILL.md` parsing, and required `aptitude.yaml` loading.
- Identity extraction: slug, version, and publish intent.
- Metadata extraction: `name`, `description`, `license`, and `compatibility` from
  `SKILL.md`; Aptitude tags, schemas, relationships, and numeric hints from
  `aptitude.yaml`.
- Security scan orchestration through NVIDIA garak.
- Anthropic-style skill validation.
- Optional performance evaluation through Hugging Face Upskill.
- Weighted ranking and `publish_decision`.
- Delivery payload assembly.
- Deterministic uploaded bundle creation with `.tar.zst`.
- Multipart registry upload to `POST /skills/{slug}`.
- One latest cache report containing normalized stage, gate, evaluator, and
  receipt evidence.

## What It Does Not Own

| Concern | Owner |
| --- | --- |
| Registry token authorization | Registry |
| Namespace and trust-tier permission checks | Registry |
| Lifecycle and review-state governance | Registry |
| Search indexing | Registry |
| Dependency solving | Resolver |
| Skill execution | Agent host / local runtime |

## Commands

```bash
# Install locally
uv venv
uv pip install -e .

# Install evaluator extras
uv pip install -e ".[evaluators]"

# Inspect only
aptitude-publisher inspect /path/to/skill

# Publish
APTITUDE_PUBLISH_TOKEN=<token> \
aptitude-publisher publish /path/to/skill --version 1.0.0

# Run the full local pipeline without upload
aptitude-publisher publish /path/to/skill --dry-run
```

The package requires Python `>=3.12`. The console script is
`aptitude-publisher`.

## Configuration

| Setting | Purpose |
| --- | --- |
| `APTITUDE_REGISTRY_URL` | Registry base URL. |
| `APTITUDE_SERVER_BASE_URL` | Alternate registry base URL fallback. |
| `APP_PORT` | Local registry port fallback. |
| `APTITUDE_PUBLISH_TOKEN` | Registry publish token. |
| `APTITUDE_INTEGRATION_PUBLISH_TOKEN` | Alternate publish token. |
| `PUBLISH_TOKEN` | Alternate publish token. |
| `PUBLISHER_GARAK_*`, `GARAK_*` | Security scan configuration. |
| `PUBLISHER_UPSKILL_*`, `UPSKILL_*` | Optional performance evaluation configuration. |

## Governance Inputs

The publisher accepts governance inputs as CLI flags and passes them to the
registry payload:

- `--trust-tier`
- `--namespace`
- `--artifact-origin`
- `--policy-pack-slug`
- `--publisher-identity`

These values are not inferred from skill content. The registry validates them
against its own service-token scopes, namespace grants, policy packs, and
trust-tier rules.

## Source Metadata

Every skill must contain a flat `aptitude.yaml` beside `SKILL.md`:

```yaml
version: "0.1.0"
intent: create_skill
tags: [python, review]
inputs_schema: {}
outputs_schema: {}
relationships:
  depends_on:
    - slug: python-testing
      version: "0.1.2"
maturity_score: 0.8
security_score: 0.9
```

`relationships` and the numeric hints `token_estimate`, `maturity_score`, and
`security_score` are optional. `name`, `description`, `license`, and
`compatibility` remain in `SKILL.md`; `agents/openai.yaml` remains an independent
OpenAI configuration file. The publisher rejects known Aptitude fields in
legacy frontmatter instead of falling back to them.

## Evaluation Report

The cache root is the absolute `XDG_CACHE_HOME` when configured, or `~/.cache`
otherwise. The report path is:

```text
<cache-root>/aptitude/publisher/<sha256(canonical-absolute-skill-directory)>.json
```

Only the latest report is retained for each canonical skill directory. Its
envelope contains `schema_version`, `skill_root`, `updated_at`, `status`,
`stages`, `gates`, `evidence`, `warnings`, `error`, and a nested signed
`inspection_receipt` when available. Raw evaluator transcripts and credentials
are not retained. Evaluator copies and working directories are temporary,
outside the source tree, and cleaned after success, failure, or timeout.

Existing `.publisher_artifacts/` directories remain preserved historical
content; they are excluded from inventory and bundles and are not read or
written by the current publisher.

MCP evaluation responses expose this location as `report_path`; clients should
not expect the removed `artifacts_dir` field.

## Reading Order

1. [Architecture](architecture.md)
2. [Evaluation Pipeline](evaluation-pipeline.md)
3. [Audit Pipeline](audit-pipeline.md)
4. [Policies](policies.md)
