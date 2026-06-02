# Aptitude Publisher - Policies

This page describes policy that the publisher enforces locally and the
governance data it passes to the registry.

## Boundary

| Question | Enforced by |
| --- | --- |
| Does the skill folder pass local discovery, identity, metadata, security, and validation gates? | Publisher |
| Does garak produce an acceptable security result? | Publisher |
| Does `SKILL.md` satisfy publisher validation rules? | Publisher |
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

The validation stage checks the skill folder and `SKILL.md` contract. Errors
block the validation gate; warnings are advisory and affect scoring only.

Blocking examples include:

- missing skill root or `SKILL.md`,
- invalid frontmatter/body shape,
- missing or invalid `name`,
- missing or invalid `description`,
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

It also blocks invalid `maturity_score` or `security_score` values outside
`0.0` to `1.0`.

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
