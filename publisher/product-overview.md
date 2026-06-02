# Aptitude Publisher - Product Overview

The Aptitude Publisher is a local CLI for skill authors and CI pipelines. It
evaluates a skill folder, records a local audit trail, builds registry metadata,
creates a deterministic `skill.tar.zst` bundle, and uploads the result to an
Aptitude Registry.

It is not a server and it does not store persistent state outside the skill
folder's `.publisher_artifacts/` directory.

## Users

| User | Goal |
| --- | --- |
| Skill author | Inspect a skill folder before publishing. |
| CI pipeline | Run the same checks and publish a versioned bundle. |
| Reviewer/operator | Inspect local evaluation artifacts and registry payload inputs. |

## What It Owns

- Skill folder inventory and `SKILL.md` parsing.
- Identity extraction: slug, version, and publish intent.
- Metadata extraction: name, description, tags, schemas, scores, and token
  estimate.
- Security scan orchestration through NVIDIA garak.
- Anthropic-style skill validation.
- Optional performance evaluation through Hugging Face Upskill.
- Weighted ranking and `publish_decision`.
- Delivery payload assembly.
- Deterministic uploaded bundle creation with `.tar.zst`.
- Multipart registry upload to `POST /skills/{slug}`.
- Local stage/gate artifacts under `.publisher_artifacts/`.

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

## Reading Order

1. [Architecture](architecture.md)
2. [Evaluation Pipeline](evaluation-pipeline.md)
3. [Audit Pipeline](audit-pipeline.md)
4. [Policies](policies.md)
