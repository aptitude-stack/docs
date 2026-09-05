# Aptitude Publisher — Audit Pipeline

> How the publisher records what ran, what passed, and what the final state of every stage and gate was.

## Overview

The publisher does not have a separate audit service. Instead, every stage and every gate writes structured records directly into the shared `PublishContext` as it runs. At the end of a pipeline run, `PublishContext` carries a complete, ordered trace of every decision made — which stages completed, which gates passed or failed, and why.

This trace is available to the CLI (which renders a human-readable report), to
the registry upload call (which can reference `security_score` and other fields
in the registry payload), and to post-run inspection of one JSON cache report.

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

## Evaluation Report

The publisher retains one latest JSON report per canonical skill directory. The
cache root is the absolute `XDG_CACHE_HOME` when configured, or `~/.cache`
otherwise. The report path is:

```text
<cache-root>/aptitude/publisher/<sha256(canonical-absolute-skill-directory)>.json
```

The report is atomically replaced with owner-only permissions. Its envelope is:

| Field | Contents |
| --- | --- |
| `schema_version` | Report schema version, currently `1`. |
| `skill_root` | Canonical absolute skill directory. |
| `updated_at` | UTC report update time. |
| `status` | `running`, `ready`, `blocked`, or `failed`. |
| `stages` | Ordered stage snapshots. |
| `gates` | Gate decisions, blocking issues, warnings, and data. |
| `evidence` | Normalized inventory, identity, metadata, security, validation, performance, and ranking evidence. |
| `warnings` | Aggregated non-blocking warnings. |
| `error` | Normalized terminal error, when a run fails. |
| `inspection_receipt` | Nested signed receipt when inspection creates one; otherwise `null`. |

The report keeps exact evaluation decisions inside `evidence`, including the
security decision and `publish_decision`. Receipt validation continues to cover
the receipt schema, MAC, expiry, configuration fingerprint, identity,
governance, and source digest.

Evaluator output is normalized before persistence. Credentials, environment
dumps, raw garak or Upskill transcripts, and temporary file paths are not
retained. Evaluators run against temporary copies and working directories
outside the source tree; those files are cleaned after success, failure, or
timeout.

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
report path      ~/.cache/aptitude/publisher/<sha256>.json

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
```

If security findings exist, each finding is printed with severity, check name, field, and evidence. If validation errors exist, they are listed in a separate section.

---

## What the Registry Receives

The registry does not receive the full publisher trace. It receives only what is embedded in the registry payload via the delivery stage:

- `metadata.security_score` — the garak score (`SecurityInfo.score`), not the finding list.
- `metadata.maturity_score` — from `aptitude.yaml`, if provided by the author.
- `governance.*` — trust tier, namespace, artifact origin, policy pack slug, and git provenance.

The finding detail, validation errors, Upskill metrics, ranking breakdown, stage history, and gate history all remain local. The registry performs its own independent governance evaluation after receiving the payload; the publisher's ranking score does not influence registry admission.

---

## Auditing the Latest Run

Read the one report for the canonical skill directory:

```bash
cat ~/.cache/aptitude/publisher/<sha256(canonical-absolute-skill-directory)>.json
```

Use `${XDG_CACHE_HOME}/aptitude/publisher/` instead when `XDG_CACHE_HOME` is
configured as an absolute path. The report's `stages`, `gates`, and `evidence`
sections contain the normalized trace and decisions. A new evaluation replaces
the previous report; there is no report history.

Existing `.publisher_artifacts/` directories are preserved historical content.
They remain excluded from inventory and bundles, and the current publisher
does not read or write them.

---

## Disabling Evaluators

Both garak and Upskill can be disabled without uninstalling them:

| Variable | Default | Effect |
| --- | --- | --- |
| `PUBLISHER_GARAK_ENABLED` | `true` | Set to `false` to skip garak. **Still blocks publish** — security has no fallback. |
| `PUBLISHER_UPSKILL_ENABLED` | `true` | Set to `false` to skip Upskill. Performance score becomes `0.0`. |

Setting `PUBLISHER_GARAK_ENABLED=false` is equivalent to garak being unconfigured: the security stage records a `block` decision and the security gate halts the pipeline. There is no way to bypass the security gate.

A cache location that resolves inside the skill directory is rejected before writing; configure `XDG_CACHE_HOME` outside the source tree.
