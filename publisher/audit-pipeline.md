# Aptitude Publisher — Audit Pipeline

> How the publisher records what ran, what passed, and what the final state of every stage and gate was.

## Overview

The publisher does not have a separate audit service. Instead, every stage and every gate writes structured records directly into the shared `PublishContext` as it runs. At the end of a pipeline run, `PublishContext` carries a complete, ordered trace of every decision made — which stages completed, which gates passed or failed, and why.

This trace is available to the CLI (which renders a human-readable report), to the registry upload call (which can reference `security_score` and other fields in the registry payload), and to post-run inspection of the `.publisher_artifacts/` directory.

---

## Trace Records

### StageSnapshot

Appended to `context.stage_history` by every stage and gate after it runs.

| Field | Type | Description |
| --- | --- | --- |
| `stage_name` | `str` | Name of the stage or gate (e.g. `"security"`, `"security_gate"`) |
| `status` | `str` | `"completed"`, `"incomplete"`, `"failed"`, `"passed"`, `"pending"` |
| `data` | `dict` | Stage-specific key/value snapshot (scores, paths, counts) |
| `messages` | `list[str]` | Human-readable notes about what the stage did or why it halted |

Stages call `context.add_snapshot(stage_name=..., status=..., data=..., messages=...)`. Gates call `context.add_snapshot(...)` immediately after `context.add_gate_result(...)`, so both records appear in `stage_history` for every gate evaluation.

### GateResult

Appended to `context.gate_history` by every gate after it runs.

| Field | Type | Description |
| --- | --- | --- |
| `gate_name` | `str` | Name of the gate (e.g. `"security_gate"`) |
| `passed` | `bool` | Whether the gate allowed the pipeline to continue |
| `blocking_issues` | `list[str]` | Issues that caused the gate to fail |
| `warnings` | `list[str]` | Non-blocking issues noted by the gate |
| `data` | `dict` | Stage-name, score, decision, counts, or other relevant context |

`blocking_issues` is empty when `passed = True`. Warnings are always informational.

---

## Per-Stage Artifacts

Each stage writes a numbered JSON artifact to `.publisher_artifacts/` inside the skill folder. These files persist after the pipeline exits and can be inspected offline.

| Artifact | Stage | Key fields |
| --- | --- | --- |
| `00_inventory.json` | Discovery | `skill_root`, `skill_markdown_path`, `script_files`, `reference_files`, `repo_url`, `commit_sha`, `notes` |
| `01_identity.json` | Identity | `slug`, `version`, `intent`, `parsed_keys`, `notes` |
| `02_metadata.json` | Metadata | `name`, `description`, `tags`, `token_estimate`, `word_count`, `repo_url`, `repo_signals`, `notes` |
| `03_ranking.json` | Ranking | `total_score`, `criteria_scores`, `weights`, `label`, `publish_decision`, `explanation` |
| `04_security.json` | Security | `scan_targets`, `checks_run`, `score`, `severity_counts`, `decision`, `findings`, `garak` (raw garak meta), `notes` |
| `05_validation.json` | Validation | `passed`, `skill_root`, `skill_file`, `checks_run`, `frontmatter_keys`, `errors`, `warnings`, `notes` |
| `05_performance_exam.json` | Performance exam | `score`, `passed`, `test_case_count`, `models_tested`, `baseline_success_rate`, `skilled_success_rate`, `skill_lift`, `token_delta`, `efficiency_label`, `upskill` (raw result meta), `notes` |
| `07_compression.json` | Compression | `algorithm`, `available`, `compressed_artifact_path`, `uncompressed_size`, `compressed_size`, `compression_ratio`, `notes` |
| `07_delivery_package.zst` | Compression | Compressed delivery payload bytes |

External evaluator artifacts are nested under:

- `garak/stdout.txt`, `garak/stderr.txt`, `garak/*.json` — raw garak output.
- `upskill/stdout.txt`, `upskill/stderr.txt`, `upskill/runs/` — raw Upskill output.

---

## CLI Report

After every `inspect` or `publish` run, the CLI prints a structured report to stdout. This is a human-readable rendering of the trace.

```
------------------------------------------------------------------------
Aptitude Publisher
------------------------------------------------------------------------
skill path      /path/to/my-skill
slug            my-skill
version         1.0.0
intent          create_skill
trust tier      untrusted
namespace       public
artifact origin internal

------------------------------------------------------------------------
Evaluation Summary
------------------------------------------------------------------------
validation      passed
security score  0.92
security gate   allow
performance     0.78
lift            0.15
token delta     -210
ranking         good
publish decision allow

------------------------------------------------------------------------
Stages
------------------------------------------------------------------------
discovery          completed
discovery_gate     passed
identity           completed
identity_gate      passed
metadata           completed
metadata_gate      passed
security           completed
security_gate      passed
validation         completed
validation_gate    passed
performance_exam   completed
ranking            completed
delivery           completed
compression        completed
```

If security findings exist, each finding is printed with severity, check name, field, and evidence. If validation errors exist, they are listed in a separate section.

---

## What the Registry Receives

The registry does not receive the full publisher trace. It receives only what is embedded in the registry payload via the delivery stage:

- `metadata.security_score` — the garak score (`SecurityInfo.score`), not the finding list.
- `metadata.maturity_score` — from frontmatter, if provided by the author.
- `governance.*` — trust tier, namespace, artifact origin, policy pack slug, and git provenance.

The finding detail, validation errors, Upskill metrics, ranking breakdown, stage history, and gate history all remain local. The registry performs its own independent governance evaluation after receiving the payload; the publisher's ranking score does not influence registry admission.

---

## Auditing a Past Run

All artifacts remain in `.publisher_artifacts/` until the next `inspect` or `publish` run overwrites them (the directory is not cleared between runs — each stage simply overwrites its numbered artifact file).

To review a past evaluation:

```bash
cat /path/to/my-skill/.publisher_artifacts/03_ranking.json
cat /path/to/my-skill/.publisher_artifacts/04_security.json
cat /path/to/my-skill/.publisher_artifacts/05_validation.json
```

The `explanation` field in `03_ranking.json` contains one sentence per criterion describing how each score was derived and what its final weighted contribution was.

---

## Disabling Evaluators

Both garak and Upskill can be disabled without uninstalling them:

| Variable | Default | Effect |
| --- | --- | --- |
| `PUBLISHER_GARAK_ENABLED` | `true` | Set to `false` to skip garak. **Still blocks publish** — security has no fallback. |
| `PUBLISHER_UPSKILL_ENABLED` | `true` | Set to `false` to skip Upskill. Performance score becomes `0.0`. |

Setting `PUBLISHER_GARAK_ENABLED=false` is equivalent to garak being unconfigured: the security stage records a `block` decision and the security gate halts the pipeline. There is no way to bypass the security gate.
