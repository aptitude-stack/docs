# Aptitude Publisher — Policies

> Governance inputs, security gate rules, validation contract, and what the publisher controls vs. what the registry enforces.

## Overview

The publisher enforces two independent policy domains:

- **Security policy** — hard gate enforced by NVIDIA garak. Cannot be bypassed. A `block` decision from garak halts the pipeline before upload.
- **Anthropic compliance** — structural and content rules for `SKILL.md`. Violations are errors that block the `ValidationGate`. Warnings are advisory.

Governance metadata (trust tier, namespace, artifact origin, etc.) is not enforced by the publisher — it is passed through to the registry, which applies its own governance checks on receive.

---

## Governance Inputs

Governance inputs are caller-provided via CLI flags. They are never derived from skill content and are not validated by the publisher beyond being placed in the delivery payload.

| Field | CLI Flag | Default | Description |
| --- | --- | --- | --- |
| `trust_tier` | `--trust-tier` | `untrusted` | Registry trust tier: `untrusted`, `internal`, or `verified`. Determines which callers can publish this skill. |
| `namespace` | `--namespace` | `public` | Target registry namespace. |
| `artifact_origin` | `--artifact-origin` | `internal` | Origin classification: `internal`, `imported`, `verified`, or `restricted`. |
| `policy_pack_slug` | `--policy-pack-slug` | `null` | Optional policy pack the skill is subject to. |
| `publisher_identity` | `--publisher-identity` | `null` | Optional identity string for provenance records (CI service account name, user, etc.). |

These values appear verbatim in the `governance` block of the registry payload. The registry validates them against its own namespace grants, trust-tier rules, and policy-pack configuration when the upload is received.

### Provenance

When the skill folder is inside a git repository, the publisher automatically includes `repo_url`, `commit_sha`, and `tree_path` in the `governance.provenance` block. Provenance is informational; the registry does not require it but records it when present.

---

## Security Policy

### Mandatory garak requirement

Security is not optional. If garak is configured, it must produce a scored result. If it does not, publishing is blocked regardless of total score or label.

| Garak status | Publisher action |
| --- | --- |
| `scored` with no critical/high findings | Decision: `allow` |
| `scored` with high findings | Decision: `review_required` |
| `scored` with critical findings | Decision: `block` → gate fails → pipeline halts |
| `not_configured` or `failed` | Decision: `block` → gate fails → pipeline halts |
| `disabled` (`PUBLISHER_GARAK_ENABLED=false`) | Decision: `block` → gate fails → pipeline halts |

There is no local fallback security check that replaces garak for publish decisions. The publisher does contain heuristic prompt-injection checks (direct injection patterns, indirect injection patterns, sensitive exfiltration requests, policy bypass patterns, dangerous action requests, hidden/obfuscated instructions, role manipulation, promptmap-style attacks, and combined risk signal detection), but these are retained solely as implementation helpers and do not affect the security score or publish decision.

### Severity thresholds

| Finding severity | Security gate effect |
| --- | --- |
| `critical` | `decision = block` — publish blocked unconditionally |
| `high` | `decision = review_required` — publish blocked unless manually overridden |
| `medium` | Contributes to score penalty (−0.15) but does not block |
| `low` | Contributes to score penalty (−0.05) but does not block |

### SecurityGate rules

The `SecurityGate` blocks the pipeline if any of the following are true:

1. `security.scanned` is `False` — garak did not run or did not produce output.
2. `security.score` is `None` — garak ran but did not produce a numeric score.
3. `security.decision` is `"block"` — critical findings or an unscored result.

A `review_required` decision does **not** block the `SecurityGate`. The pipeline continues, but the `RankingStage` sets `publish_decision = "review_required"`, which the CLI prints and which prevents the upload step from proceeding (the CLI checks `publish_decision != "block"` before uploading, but `review_required` is treated the same as `block` at upload time).

---

## Anthropic Compliance Policy

The `ValidationStage` enforces the Anthropic SKILL.md contract. All rules are applied to the skill folder and `SKILL.md`. Violations produce errors (blocking) or warnings (advisory).

### Error rules (blocking)

| Rule | Requirement |
| --- | --- |
| Skill root exists | Directory must exist |
| Skill folder kebab-case | Folder name matches `[a-z0-9]+(-[a-z0-9]+)*` |
| SKILL.md present | `SKILL.md` must exist in the skill root |
| No README in skill folder | `README.md` must not appear inside the skill folder |
| YAML frontmatter present | `SKILL.md` must start with `---` delimited YAML block |
| `name` field present | Non-empty `name` in frontmatter |
| `name` kebab-case | No spaces, capital letters, or special characters |
| `name` matches folder | `frontmatter.name` must match the folder name |
| `name` reserved words | `"claude"` and `"anthropic"` are banned |
| `description` present | Non-empty `description` in frontmatter |
| `description` length | Under 1024 characters |
| `description` trigger guidance | Must contain action markers (e.g. "creates", "handles") and trigger markers (e.g. "use when", "when user") |
| No XML angle brackets | `<` or `>` forbidden in any frontmatter string field |
| `compatibility` length | 1–500 characters if the field is present |
| Body present | SKILL.md must contain non-empty content after frontmatter |

### Warning rules (advisory)

| Rule | Guidance |
| --- | --- |
| Instructions heading | Body should include a `# Instructions` heading |
| Examples presence | Body should include at least one example |
| Troubleshooting presence | Body should include a troubleshooting section |

### ValidationGate rules

The `ValidationGate` blocks if:

1. No validation artifact was written.
2. No checks were recorded as run.
3. Any validation error is present (`validation.errors` is non-empty).

Warnings in `validation.warnings` propagate to the ranking stage, where they reduce the `anthropic_compliance` score from `1.0` to `0.8`. They do not block the gate.

---

## Ranking and the Publish Decision

The ranking stage synthesizes the security and validation outcomes into a final `publish_decision`. The decision overrides the numeric score in determining whether an upload proceeds.

| Condition | `publish_decision` |
| --- | --- |
| `security.decision == "block"` | `block` |
| `not validation.passed` | `review_required` |
| `security.decision == "review_required"` | `review_required` |
| All gates passed, no high/critical security | `allow` |

The `publish` CLI command uploads only when `publish_decision == "allow"`. Both `block` and `review_required` halt the upload.

---

## Registry vs. Publisher Policy Boundary

The publisher and the registry each enforce distinct policy domains. Neither delegates to the other.

| Policy question | Enforced by |
| --- | --- |
| Is the skill content free of injection attacks? | Publisher (garak) |
| Does SKILL.md follow Anthropic's structure rules? | Publisher (validation) |
| Is the publisher's `trust_tier` allowed for this namespace? | Registry |
| Does the registry caller have `publish` scope? | Registry |
| Is the lifecycle state a valid transition? | Registry |
| Is the bundle checksum correct? | Registry (on receive) |
| Are governance policy pack rules satisfied? | Registry |

The publisher cannot grant or override registry governance decisions. A skill that passes all publisher gates may still be rejected by the registry if the caller's token lacks `publish` scope, if the trust tier is not permitted for the target namespace, or if a policy pack rule blocks it.

---

## Configuration Reference

| Environment variable | Default | Description |
| --- | --- | --- |
| `APTITUDE_REGISTRY_URL` | `http://127.0.0.1:8000` | Registry base URL |
| `APTITUDE_PUBLISH_TOKEN` | — | Registry publish token |
| `PUBLISHER_GARAK_ENABLED` | `true` | Set `false` to skip garak (still blocks publish) |
| `PUBLISHER_GARAK_COMMAND` | — | Full garak command template (`{skill_path}`, `{skill_file}`, `{artifact_dir}`) |
| `GARAK_TARGET_TYPE` | — | Garak target type (e.g. `openai`) |
| `GARAK_TARGET_NAME` | — | Garak target name (e.g. `gpt-4o-mini`) |
| `GARAK_PROBES` | `promptinject` | Garak probe set |
| `GARAK_DETECTORS` | — | Garak detector override |
| `PUBLISHER_GARAK_TIMEOUT_SECONDS` | `180` | Garak subprocess timeout |
| `PUBLISHER_UPSKILL_ENABLED` | `true` | Set `false` to skip Upskill |
| `PUBLISHER_UPSKILL_COMMAND` | — | Full Upskill command template (`{skill_path}`, `{artifact_dir}`, `{runs_dir}`) |
| `UPSKILL_MODELS` | — | Comma-separated model names for Upskill |
| `UPSKILL_PROVIDER` | — | Upskill provider name |
| `UPSKILL_BASE_URL` | — | OpenAI-compatible base URL for direct Upskill API mode |
| `UPSKILL_API_KEY` | — | API key for direct Upskill API mode |
| `UPSKILL_TESTS_PATH` | — | Path to JSON test cases for direct Upskill API mode |
| `UPSKILL_NO_BASELINE` | `false` | Skip baseline measurement in Upskill |
| `PUBLISHER_UPSKILL_TIMEOUT_SECONDS` | `600` | Upskill subprocess timeout |
