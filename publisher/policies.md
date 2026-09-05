# Aptitude Publisher - Policies

This page describes policy that the publisher enforces locally and the
governance data it passes to the registry.

## Boundary

| Question | Enforced by |
| --- | --- |
| Does the skill folder pass local discovery, identity, metadata, security, and validation gates? | Publisher |
| Does garak produce an acceptable security result? | Publisher |
| Do `SKILL.md` and `aptitude.yaml` satisfy publisher validation rules? | Publisher |
| Is this caller allowed to publish to a namespace/trust tier? | Registry |
| Is the service token valid and scoped for publish? | Registry |
| Is lifecycle/review/promotion state valid? | Registry |
| Is the uploaded bundle stored with correct checksums? | Registry |

The publisher cannot grant registry permissions and cannot override registry
governance.

## Governance Inputs

The publisher accepts these caller-provided values:

| Field | CLI flag | Default |
| --- | --- | --- |
| `trust_tier` | `--trust-tier` | `untrusted` |
| `namespace` | `--namespace` | `public` |
| `artifact_origin` | `--artifact-origin` | `internal` |
| `policy_pack_slug` | `--policy-pack-slug` | unset |
| `publisher_identity` | `--publisher-identity` | unset |

These values are included in the registry payload. They are not inferred from
skill content.

When git provenance is available, the publisher also includes repository URL,
commit SHA, and tree path as provenance data.

## Security Policy

NVIDIA garak is the authoritative security source for publish decisions.

| Garak result | Publisher effect |
| --- | --- |
| Scored, no high/critical findings | security decision `allow`; pipeline continues |
| Scored, high findings | security decision `review_required`; pipeline continues |
| Scored, critical findings | security decision `block`; security gate fails |
| Not configured, disabled, failed, or unscored | security decision `block`; security gate fails |

`SecurityGate` blocks when:

- the scan did not run,
- no numeric score was produced,
- the decision is invalid,
- the decision is `block`.

`review_required` does not fail `SecurityGate`.

## Validation Policy

The validation stage checks the skill folder, `SKILL.md` contract, and required
`aptitude.yaml` sidecar. Errors block the validation gate; warnings are
advisory and affect scoring only.

Blocking examples include:

- missing skill root or `SKILL.md`,
- missing or invalid `aptitude.yaml`,
- invalid frontmatter/body shape,
- missing or invalid `name`,
- missing or invalid `description`,
- known Aptitude fields left in legacy `SKILL.md` frontmatter,
- `README.md` inside the skill folder,
- reserved names,
- invalid compatibility field,
- empty body.

## Metadata Gate

`MetadataGate` blocks when required metadata is missing:

- `name`,
- `description`,
- `tags`,
- `inputs_schema`,
- `outputs_schema`.

`name` and `description` come from `SKILL.md`; `tags`, `inputs_schema`, and
`outputs_schema` come from `aptitude.yaml`. The sidecar also carries
`version`, `intent`, and optional `relationships`.

It also blocks invalid `maturity_score` or `security_score` values outside
`0.0` to `1.0`.

## Source Metadata Policy

`aptitude.yaml` is a flat YAML mapping beside `SKILL.md` with these required
fields:

```yaml
version: "0.1.0"
intent: create_skill
tags: [python]
inputs_schema: {}
outputs_schema: {}
relationships:
  depends_on:
    - slug: python-testing
      version: "0.1.2"
```

`relationships` is optional; omitted relationship families default to empty
lists. Optional numeric hints are `token_estimate`, `maturity_score`, and
`security_score`. Computed evaluation values supersede authored hints when
available. CLI and MCP values override sidecar identity values, while the
`SKILL.md` `name` supplies the slug by default.

The publisher rejects duplicate keys, unknown sidecar fields, invalid types,
and malformed relationship selectors. It rejects known Aptitude fields in
legacy frontmatter rather than falling back to them. Move those fields manually
to `aptitude.yaml`; keep `name`, `description`, `license`, and `compatibility` in
`SKILL.md`. `agents/openai.yaml` remains independent and unchanged.

## Evaluation Report Policy

The publisher keeps one latest JSON report for each canonical skill directory.
The cache root is the absolute `XDG_CACHE_HOME` when configured, or `~/.cache`
otherwise:

```text
<cache-root>/aptitude/publisher/<sha256(canonical-absolute-skill-directory)>.json
```

The report is atomically replaced with owner-only permissions. It contains
`schema_version`, `skill_root`, `updated_at`, `status`, `stages`, `gates`,
`evidence`, `warnings`, `error`, and a nested signed `inspection_receipt` when
available. Status values are `running`, `ready`, `blocked`, and `failed`.
Normalized stage, gate, security, performance, and decision evidence is kept;
credentials, raw evaluator transcripts, environment dumps, and temporary paths
are not retained. Evaluator copies and working directories live outside the
source tree and are cleaned after success, failure, or timeout.

Existing `.publisher_artifacts/` directories are preserved historical content,
excluded from inventory and bundles, and not read or written. The MCP response
field is `report_path`; `artifacts_dir` is obsolete.

## Publish Decision

The ranking stage computes `publish_decision`:

| Condition | Decision |
| --- | --- |
| `security.decision == "block"` | `block` |
| validation did not pass | `review_required` |
| `security.decision == "review_required"` | `review_required` |
| otherwise | `allow` |

Current CLI behavior blocks upload only when `publish_decision == "block"`.
`review_required` is visible in output but is not currently an upload stop in
`publisher/app/cli.py`.

## Environment Reference

| Variable | Purpose |
| --- | --- |
| `APTITUDE_REGISTRY_URL` | Registry base URL. |
| `APTITUDE_SERVER_BASE_URL` | Alternate registry base URL. |
| `APP_PORT` | Local registry port fallback. |
| `APTITUDE_PUBLISH_TOKEN` | Registry publish token. |
| `APTITUDE_INTEGRATION_PUBLISH_TOKEN` | Alternate publish token. |
| `PUBLISH_TOKEN` | Alternate publish token. |
| `PUBLISHER_GARAK_ENABLED` | Disabling garak still blocks publish. |
| `PUBLISHER_GARAK_COMMAND` | garak command template. |
| `GARAK_TARGET_TYPE`, `GARAK_TARGET_NAME` | garak target configuration. |
| `GARAK_PROBES`, `GARAK_DETECTORS` | garak probe/detector configuration. |
| `PUBLISHER_GARAK_TIMEOUT_SECONDS` | garak timeout. |
| `PUBLISHER_UPSKILL_ENABLED` | Enable/disable optional Upskill evaluation. |
| `PUBLISHER_UPSKILL_COMMAND` | Upskill command template. |
| `UPSKILL_MODELS`, `UPSKILL_PROVIDER`, `UPSKILL_BASE_URL`, `UPSKILL_API_KEY`, `UPSKILL_TESTS_PATH`, `UPSKILL_NO_BASELINE` | Upskill configuration. |
| `PUBLISHER_UPSKILL_TIMEOUT_SECONDS` | Upskill timeout. |
