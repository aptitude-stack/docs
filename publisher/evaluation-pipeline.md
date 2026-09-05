# Aptitude Publisher — Evaluation Pipeline

> Stage-by-stage breakdown of what runs, what each gate checks, and how the publish decision is reached.

## Overview

The evaluation pipeline transforms a skill folder into a publish decision and a
registry-ready bundle. It runs eight sequential stages interleaved with six
gates. A terminal gate failure halts the pipeline immediately; evaluator gate
results may be recorded before ranking continues.

```mermaid
flowchart TD
    IN["Skill folder path\n+ CLI flags"]
    D["Stage 0: Discovery\nInventory + source parsing"]
    DG{"DiscoveryGate"}
    I["Stage 1: Identity\nSlug · version · intent"]
    IG{"IdentityGate"}
    M["Stage 2: Metadata\nSKILL.md + aptitude.yaml"]
    MG{"MetadataGate"}
    S["Stage 3: Security\nNVIDIA garak scan"]
    SG{"SecurityGate"}
    V["Stage 4: Validation\nAnthropic SKILL.md compliance"]
    VG{"ValidationGate"}
    P["Stage 5: Performance Exam\nHugging Face Upskill"]
    PG{"PerformanceExamGate"}
    R["Stage 6: Ranking\nWeighted quality score + publish decision"]
    DEL["Stage 7: Delivery\nRegistry payload assembly"]
    OUT["Latest cache report\nstages + gates + evidence"]

    HALT["Pipeline halted\n(gate failure)"]

    IN --> D --> DG
    DG -- pass --> I --> IG
    DG -- fail --> HALT
    IG -- pass --> M --> MG
    IG -- fail --> HALT
    MG -- pass --> S --> SG
    MG -- fail --> HALT
    SG -- pass --> V --> VG
    SG -- fail --> HALT
    VG -- pass --> P --> PG
    PG -- pass --> R --> DEL --> OUT
    PG -- fail --> R
    VG -- fail --> HALT
```

---

## Stage 0 — Discovery

**Module:** `stages/discovery.py`  
**Report evidence:** `evidence.inventory`

Reads the skill folder, inventories all files, parses `SKILL.md` into
frontmatter and body, and loads the required `aptitude.yaml` sidecar.

What it does:

- Resolves the skill root directory from the provided path (accepting a folder path, a `SKILL.md` path, or any file inside the skill folder).
- Classifies every file under the root into: `SKILL.md` (primary),
  `aptitude.yaml`, `agents/openai.yaml` when present, `scripts/`, `references/`,
  `assets/`, companion markdown files, and uncategorized other files.
- Discovers git provenance: `repo_root`, `repo_url`, `commit_sha`, `tree_path` (via `git` subprocess calls).
- Parses `SKILL.md` frontmatter and body, and the flat `aptitude.yaml` mapping,
  into `context.source.parsed_content`.
- Checkpoints the inventory in the latest cache report.

**DiscoveryGate** blocks if:
- Skill root does not exist or is not a directory.
- `SKILL.md` was not found.
- `parsed_content` is missing, frontmatter or sidecar is invalid, or body is not
  a string.
- The inventory result cannot be recorded in the cache report.

---

## Stage 1 — Identity

**Module:** `stages/identity.py`  
**Report evidence:** `evidence.identity`

Extracts the three registry identity fields: `slug`, `version`, and `intent`.

Sources, in order of precedence:
- CLI overrides (`--slug`, `--version`, `--intent`).
- `SKILL.md` frontmatter: `name` (→ slug).
- `aptitude.yaml`: `version` and `intent`.

**IdentityGate** blocks if any of `slug`, `version`, or `intent` is absent.

---

## Stage 2 — Metadata

**Module:** `stages/metadata.py`  
**Report evidence:** `evidence.metadata`

Combines standard skill fields from `SKILL.md` with Aptitude publishing fields
from the adjacent `aptitude.yaml`, then computes publisher-side fields.

Fields and sources:

| Field | Source |
| --- | --- |
| `name` | `SKILL.md` `frontmatter.name` |
| `description` | `SKILL.md` `frontmatter.description` |
| `license` | `SKILL.md` `frontmatter.license`, when present |
| `compatibility` | `SKILL.md` `frontmatter.compatibility`, when present |
| `tags` | `aptitude.yaml` `tags` |
| `inputs_schema` | `aptitude.yaml` `inputs_schema` |
| `outputs_schema` | `aptitude.yaml` `outputs_schema` |
| `token_estimate` | Authored hint from `aptitude.yaml`; publisher-computed value is used for evaluation when available |
| `maturity_score` | Authored hint from `aptitude.yaml`, when present |
| `security_score` | Authored hint from `aptitude.yaml`, when present; authoritative security evidence supersedes it |

Publisher-computed fields (not authored in `SKILL.md` or `aptitude.yaml`):

- **`token_estimate`** — content heuristic:
  `max(1, int(round(max(len(text) / 4, word_count * 1.3))))`. This is a
  provisional estimate; an authored sidecar hint is retained for traceability,
  and it is replaced by
  `performance_exam.skilled_avg_tokens` when Upskill runs successfully.
- **`word_count`** — counted from the SKILL.md body plus companion markdown files.

Optional enrichment: GitHub repository signals (star count, forks, etc.) are
fetched via `github_api.fetch_repository_signals` and stored in
`metadata.extra.repo_signals` if a repo URL was discovered. Relationships are
read from the optional `aptitude.yaml` `relationships` mapping and normalized
for delivery.

**MetadataGate** blocks if `name`, `description`, `tags`, `inputs_schema`, or
`outputs_schema` is absent. It also blocks if `maturity_score` or
`security_score` is present but outside `0.0` to `1.0`.

---

## Stage 3 — Security

**Module:** `stages/security.py`  
**Report evidence:** `evidence.security`
**Integration:** `integrations/garak_security.py`

NVIDIA garak is the sole authoritative security source. Local heuristic checks exist in the code as implementation helpers but do not contribute to the security score or the publish decision.

### Garak execution

The garak command is built from:
1. `PUBLISHER_GARAK_COMMAND` environment variable (a template supporting
   `{skill_path}`, `{skill_file}`, and a temporary `{artifact_dir}`).
2. Auto-constructed command using `GARAK_TARGET_TYPE`, `GARAK_TARGET_NAME`, and optionally `GARAK_PROBES` (default `promptinject`) and `GARAK_DETECTORS`.

Timeout: `PUBLISHER_GARAK_TIMEOUT_SECONDS` (default 180 seconds).

### Garak result normalization

Garak output is parsed into a flat finding list before the temporary evaluator
workspace is removed. Each normalized finding has `check`, `severity` (`low` /
`medium` / `high` / `critical`), `field`, `reason`, and `evidence`.

Score is computed from findings using the penalty table:

| Severity | Penalty |
| --- | --- |
| `low` | −0.05 |
| `medium` | −0.15 |
| `high` | −0.30 |
| `critical` | −0.50 |

Starting score is `1.0`; penalties are summed and clamped to `[0.0, 1.0]`. If garak returns an authoritative numeric score, that value overrides the penalty-derived score.

### Security decision

| Condition | Decision |
| --- | --- |
| Any `critical` finding | `block` |
| Any `high` finding | `review_required` |
| No high/critical findings | `allow` |

If garak is not configured or does not produce a scored result, `decision = block` with a synthetic critical finding (`garak:required_scan`).

**SecurityGate** blocks if:
- `security.scanned` is false (garak did not run or did not score).
- `security.score` is `None`.
- `security.decision` is `block`.

---

## Stage 4 — Validation

**Module:** `stages/validation.py`  
**Report evidence:** `evidence.validation`

Validates the skill folder and `SKILL.md` against the Anthropic skill-writing contract. Produces blocking errors and non-blocking warnings.

### Checks run

| Check | Type | Rule |
| --- | --- | --- |
| `skill_root_exists` | error | Skill root exists and is a directory |
| `skill_folder_kebab_case` | error | Folder name matches `[a-z0-9]+(-[a-z0-9]+)*` |
| `skill_md_present` | error | `SKILL.md` exists in the skill root |
| `readme_absent_in_skill_folder` | error | `README.md` must not appear inside the skill folder |
| `yaml_frontmatter_present` | error | `SKILL.md` starts with `---` YAML block |
| `aptitude_manifest_present` | error | `aptitude.yaml` exists beside `SKILL.md` and parses as a mapping |
| `frontmatter_name_present` | error | `name` field is non-empty |
| `frontmatter_name_kebab_case` | error | `name` is kebab-case, lowercase and hyphens only |
| `frontmatter_name_matches_folder` | error | `name` matches the skill folder name |
| `frontmatter_name_reserved_words` | error | `name` does not contain `"claude"` or `"anthropic"` |
| `frontmatter_description_present` | error | `description` is non-empty |
| `frontmatter_description_length` | error | Description is under 1024 characters |
| `frontmatter_description_trigger_guidance` | error | Description includes action markers + trigger markers (heuristic for use-when guidance) |
| `frontmatter_no_xml_angle_brackets` | error | No `<` or `>` in any frontmatter string field |
| `no_legacy_aptitude_frontmatter` | error | Known Aptitude fields are absent from top-level or nested frontmatter |
| `compatibility_length_if_present` | error | `compatibility` is 1–500 characters if present |
| `body_present` | error | Body is non-empty after frontmatter |
| `body_instructions_heading` | warning | Body contains `# Instructions` heading |
| `body_examples_presence` | warning | Body contains the word "example" |
| `body_troubleshooting_presence` | warning | Body contains the word "troubleshooting" |

Errors set `validation.passed = False`. Warnings are informational and do not block.

**ValidationGate** blocks if `validation.errors` is non-empty.

---

## Stage 5 — Performance Exam

**Module:** `stages/performance_exam.py`  
**Report evidence:** `evidence.performance`
**Integration:** `integrations/upskill_eval.py`

Hugging Face Upskill is the sole performance measurement source. Missing or
failed evidence blocks the performance gate; inconclusive evidence remains
available for manual review. The publisher does not produce its own performance
estimate when Upskill is unavailable.

### Upskill execution modes

1. **Direct Python API** — used when `UPSKILL_BASE_URL` + `UPSKILL_API_KEY` + `UPSKILL_TESTS_PATH` are all set. Calls `evaluate_skill()` from the `upskill` package with a custom OpenAI-compatible provider.
2. **Subprocess CLI** — builds an `upskill eval <skill_path>` command from `PUBLISHER_UPSKILL_COMMAND` template or from auto-detected executable + environment variables.

Both modes use a temporary skill copy and temporary working directory outside
the source tree. The evaluator workspace is cleaned after success, failure, or
timeout; only normalized evidence is placed in the latest report.

Timeout: `PUBLISHER_UPSKILL_TIMEOUT_SECONDS` (default 600 seconds).

### Metrics captured

| Field | Description |
| --- | --- |
| `score` | Composite score: `(skilled_success_rate × 0.70) + (lift × 0.20) + (token_score × 0.10)` |
| `passed` | Whether the skill is beneficial (skilled success ≥ baseline) |
| `test_case_count` | Number of test cases run |
| `baseline_success_rate` | Fraction of test cases passing without the skill |
| `skilled_success_rate` | Fraction of test cases passing with the skill |
| `skill_lift` | `skilled_success_rate - baseline_success_rate` |
| `baseline_avg_tokens` | Average tokens per test case without the skill |
| `skilled_avg_tokens` | Average tokens per test case with the skill |
| `token_delta` | `skilled_avg_tokens - baseline_avg_tokens` (negative = improvement) |
| `efficiency_label` | `"improved"` if `token_delta < 0`, else `"neutral"` |

When Upskill produces a positive `skilled_avg_tokens`, that value replaces the
content-heuristic `metadata.token_estimate` for the remainder of the pipeline.

If Upskill is unavailable or produces no score, `performance_exam.score = None` and the performance criterion contributes `0.0` to ranking.

---

## Stage 6 — Ranking

**Module:** `stages/ranking.py`  
**Report evidence:** `evidence.ranking`

Combines all upstream evaluation signals into a single weighted quality score and a final `publish_decision`.

### Scoring rubric

| Criterion | Weight | Source |
| --- | --- | --- |
| `security` | 0.30 | `security.score` (from garak) |
| `performance_exam` | 0.25 | `performance_exam.score` (from Upskill) |
| `anthropic_compliance` | 0.20 | Validation errors/warnings (see below) |
| `token_efficiency` | 0.15 | Upskill token delta (preferred) or metadata token estimate |
| `metadata_completeness` | 0.10 | Fraction of required metadata fields present |
| `instruction_quality` | 0.00 | SKILL.md body structure (currently zero-weighted) |

### Criterion scoring details

**`anthropic_compliance`:** `1.0` if no errors and no warnings, `0.8` if warnings only, `0.0` if any errors.

**`token_efficiency`** — two paths:

If Upskill ran and produced token data:
| `skilled_avg_tokens` vs baseline | Score |
| --- | --- |
| ≤ 70% of baseline | 1.0 |
| ≤ 85% of baseline | 0.8 |
| ≤ 100% of baseline | 0.65 |
| > baseline | 0.3 |

If only the content-heuristic `token_estimate` is available:
| Token estimate | Score |
| --- | --- |
| ≤ 200 | 1.0 |
| ≤ 500 | 0.8 |
| ≤ 1000 | 0.6 |
| ≤ 2000 | 0.4 |
| > 2000 | 0.2 |
| `None` | 0.4 |

**`metadata_completeness`:** Fraction of `[name, description, tags, inputs_schema, outputs_schema]` that are present (0.0 – 1.0).

### Total score and label

```
total_score = Σ (weight × criterion_score)
```

| Range | Label |
| --- | --- |
| ≥ 0.85 | `excellent` |
| ≥ 0.70 | `good` |
| ≥ 0.55 | `review` |
| < 0.55 | `poor` |

### Publish decision

The `publish_decision` is determined independently of the label, using gate results:

| Condition | Decision |
| --- | --- |
| `security.decision == "block"` | `block` (security overrides everything) |
| `not validation.passed` or `security.decision == "review_required"` | `review_required` |
| Otherwise | `allow` |

A `block` decision always prevents upload regardless of total score.

---

## Stage 7 — Delivery

**Module:** `stages/delivery.py`  
**Report record:** delivery completion is recorded in `stages`; the upload
receives the final payload in memory.

Assembles the `DeliveryPayload` — the registry-ready JSON structure that becomes the `metadata` part of the multipart upload.

Payload shape:

```json
{
  "intent": "create_skill",
  "version": "1.0.0",
  "metadata": {
    "name": "...",
    "description": "...",
    "tags": [...],
    "inputs_schema": {...},
    "outputs_schema": {...},
    "token_estimate": 420,
    "maturity_score": null,
    "security_score": 0.95
  },
  "governance": {
    "trust_tier": "untrusted",
    "namespace": "public",
    "artifact_origin": "internal",
    "policy_pack_slug": null,
    "provenance": {
      "repo_url": "https://github.com/...",
      "commit_sha": "abc123",
      "tree_path": "skills/my-skill",
      "publisher_identity": null
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

Provenance is included only if `repo_url` and `commit_sha` were discovered during inventory.

---

## Persisted Report

The pipeline checkpoints after each completed stage and gate into one latest
JSON report per canonical skill directory. The cache root is the absolute
`XDG_CACHE_HOME` when configured, or `~/.cache` otherwise:

```text
<cache-root>/aptitude/publisher/<sha256(canonical-absolute-skill-directory)>.json
```

The report envelope contains `schema_version`, `skill_root`, `updated_at`,
`status`, `stages`, `gates`, `evidence`, `warnings`, `error`, and nested
`inspection_receipt`. Status is `running`, `ready`, `blocked`, or `failed`.
Writes use a same-directory temporary file and atomic replacement with
owner-only permissions. A new run clears any previous receipt before writing;
corrupt or expired cached evidence causes reevaluation.

The report stores normalized diagnostics and evaluator evidence only. It does
not retain credentials, raw evaluator transcripts, environment dumps, or dead
temporary paths. Existing `.publisher_artifacts/` directories remain preserved
historical content, are excluded from inventory and bundles, and are not read or
written.
