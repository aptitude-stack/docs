# Aptitude Publisher — Architecture

> Package map, layer breakdown, pipeline data flow, and registry transport contract.

## System Context

The publisher is a local Python process. It has no server component and no persistent state. Each invocation runs the full pipeline from scratch and writes outputs into a `.publisher_artifacts/` directory inside the skill folder.

```
┌───────────────────────────────────────────────────────────────────┐
│                     Aptitude Publisher                            │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                     CLI  (app/cli.py)                       │  │
│  │       inspect · publish · --dry-run · --trust-tier …        │  │
│  └──────────────────────────────┬──────────────────────────────┘  │
│                                 │                                  │
│  ┌──────────────────────────────▼──────────────────────────────┐  │
│  │               Pipeline  (app/pipeline.py)                   │  │
│  │  PublisherPipeline.run() — executes stages in order,        │  │
│  │  checks one gate per stage, halts on gate failure.          │  │
│  └──────────┬──────────────────────────────────────────────────┘  │
│             │                                                      │
│  ┌──────────▼──────────────────────────────────────────────────┐  │
│  │                      Stages                                  │  │
│  │  discovery · identity · metadata · security · validation    │  │
│  │  performance_exam · ranking · delivery · compression        │  │
│  └──────────┬──────────────────────────────────────────────────┘  │
│             │                                                      │
│  ┌──────────▼──────────────────────────────────────────────────┐  │
│  │                       Gates                                  │  │
│  │  discovery_gate · identity_gate · metadata_gate             │  │
│  │  security_gate · validation_gate                            │  │
│  └──────────┬──────────────────────────────────────────────────┘  │
│             │                                                      │
│  ┌──────────▼──────────────────────────────────────────────────┐  │
│  │                   Integrations                               │  │
│  │  garak_security · upskill_eval · github_api · external_tools│  │
│  └──────────┬──────────────────────────────────────────────────┘  │
│             │                                                      │
│  ┌──────────▼──────────────────────────────────────────────────┐  │
│  │                  Domain  (domain/models.py)                  │  │
│  │  PublishContext · SkillSource · SkillInventory · ...        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│             │                                                      │
│  ┌──────────▼──────────────────────────────────────────────────┐  │
│  │               Artifacts  (artifacts/bundle.py)               │  │
│  │   build_bundle_bytes() — tar.zst skill archive              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│             │                                                      │
│  ┌──────────▼──────────────────────────────────────────────────┐  │
│  │             Registry Client  (registry/client.py)            │  │
│  │  publish_to_registry() — multipart POST /skills/{slug}      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬─────────────────────────────┘
                                      │  HTTP POST multipart/form-data
                          ┌───────────▼──────────────────┐
                          │    Aptitude Registry Server  │
                          └──────────────────────────────┘
```

## Package Map

```
publisher/
  publisher/
    app/
      cli.py            # CLI entrypoint: inspect + publish commands
      pipeline.py       # PublisherPipeline — stage ordering and gate dispatch
      __init__.py
    artifacts/
      bundle.py         # build_bundle_bytes() — skill folder → .tar.zst bytes
    domain/
      models.py         # PublishContext, SkillSource, SkillInventory, all stage info models,
                        # StageSnapshot, GateResult
    gates/
      base.py           # PublisherGate abstract base class
      discovery.py      # DiscoveryGate — SKILL.md exists and inventory is complete
      identity.py       # IdentityGate — slug, version, intent are present
      metadata.py       # MetadataGate — name, description, tags are present
      security.py       # SecurityGate — garak ran and did not block
      validation.py     # ValidationGate — Anthropic checks passed
    integrations/
      external_tools.py    # run_command, resolve_executable, render_command, configured_bool
      garak_security.py    # run_garak_security_scan, GarakSecurityResult
      upskill_eval.py      # run_upskill_evaluation, UpskillEvaluation
      github_api.py        # fetch_repository_signals — repo signals for metadata extra
    registry/
      client.py         # publish_to_registry, build_publish_metadata, RegistryPublishResult
                        # multipart form encoding
    stages/
      base.py           # PublisherStage abstract base class
      discovery.py      # Stage 0: inventory + SKILL.md parsing
      identity.py       # Stage 1: slug, version, intent extraction
      metadata.py       # Stage 2: metadata extraction + token estimate
      security.py       # Stage 3: garak security scan
      validation.py     # Stage 4: Anthropic SKILL.md compliance checks
      performance_exam.py # Stage 5: Upskill performance evaluation
      ranking.py        # Stage 6: weighted quality scoring + publish decision
      delivery.py       # Stage 7: registry payload assembly
      compression.py    # Stage 8: Zstandard bundle compression
```

## Layer Breakdown

### CLI (`app/cli.py`)

Entry point for humans. Parses arguments, creates a `PublishContext` via `PublisherPipeline.create_context()`, calls `pipeline.run()`, prints a structured report, and (for `publish`) calls `build_bundle_bytes()` then `publish_to_registry()`.

Governance inputs (`trust_tier`, `namespace`, `artifact_origin`, `policy_pack_slug`, `publisher_identity`) are parsed here from CLI flags and stored in `SkillSource`. They are never derived from skill content.

### Pipeline (`app/pipeline.py`)

`PublisherPipeline` holds an ordered tuple of `PublisherStage` instances and a map of `PublisherGate` instances keyed by stage name. `run()` iterates the stages in order. After each stage, if a gate is registered for that stage, it runs `gate.verify(context)`. If a gate fails, the loop breaks and no subsequent stage runs.

Gates are registered for: `discovery`, `identity`, `metadata`, `security`, `validation`. No gates are registered for `performance_exam`, `ranking`, `delivery`, or `compression` — those stages always run if the earlier gates pass.

### Domain (`domain/models.py`)

`PublishContext` is the mutable state object threaded through every stage and gate. It carries:

- `source` (`SkillSource`) — the original input: file path, governance overrides, raw content.
- `inventory` (`SkillInventory`) — discovered folder structure, file lists, git provenance.
- `identity` (`IdentityInfo`) — slug, version, intent.
- `metadata` (`MetadataInfo`) — name, description, tags, schemas, token estimate.
- `security` (`SecurityInfo`) — garak score, findings, severity counts, decision.
- `validation` (`ValidationInfo`) — passed flag, errors, warnings, checks run.
- `performance_exam` (`PerformanceExamInfo`) — Upskill metrics: lift, success rates, token delta.
- `ranking` (`RankingInfo`) — weighted scores, label, publish decision.
- `delivery_payload` (`DeliveryPayload`) — registry-ready slug, version, content, metadata, governance, relationships.
- `compression` (`CompressionInfo`) — algorithm, sizes, ratio, artifact paths.
- `stage_history` (`list[StageSnapshot]`) — one entry per stage run.
- `gate_history` (`list[GateResult]`) — one entry per gate run.

### Stages (`stages/`)

Each stage is a `PublisherStage` subclass with a `name` string and a `run(context)` method. Stages mutate `context` in place and call `context.add_snapshot(...)` to record their result in `stage_history`. Stages also write a numbered JSON artifact to `.publisher_artifacts/`.

| Stage | Artifact | Summary |
| --- | --- | --- |
| `discovery` | `00_inventory.json` | Skill folder inventory + SKILL.md parsing |
| `identity` | `01_identity.json` | Slug, version, intent |
| `metadata` | `02_metadata.json` | Metadata fields + heuristic token estimate |
| `security` | `04_security.json` | Garak score, findings, decision |
| `validation` | `05_validation.json` | Errors, warnings, checks run |
| `performance_exam` | `05_performance_exam.json` | Upskill metrics |
| `ranking` | `03_ranking.json` | Weighted score, label, publish decision |
| `delivery` | _(in-memory only)_ | Registry payload shape |
| `compression` | `07_compression.json` + `07_delivery_package.zst` | Compressed bundle |

### Gates (`gates/`)

Each gate is a `PublisherGate` subclass that reads from `context` only — it does not modify stage outputs. Gates call `context.add_gate_result(...)` and `context.add_snapshot(...)` so their outcome appears in both `gate_history` and `stage_history` for full traceability.

| Gate | Registered after | Blocks on |
| --- | --- | --- |
| `DiscoveryGate` | `discovery` | SKILL.md missing, inventory artifact missing, bad frontmatter |
| `IdentityGate` | `identity` | slug, version, or intent is absent |
| `MetadataGate` | `metadata` | name, description, or tags is absent |
| `SecurityGate` | `security` | garak did not produce a score, or decision is `block` |
| `ValidationGate` | `validation` | any validation error present |

### Integrations (`integrations/`)

Thin wrappers around external processes:

- **`garak_security.py`** — builds a `garak` command from environment variables or a command template, runs it as a subprocess, parses JSON/JSONL report artifacts, and normalizes findings into a `GarakSecurityResult`.
- **`upskill_eval.py`** — runs Upskill via the Python API (when a custom `UPSKILL_BASE_URL` is set) or as a subprocess, parses the text-table or JSON output, and normalizes results into an `UpskillEvaluation`.
- **`github_api.py`** — fetches optional repository signals (star count, fork count, etc.) from the GitHub API to enrich metadata extra fields.
- **`external_tools.py`** — shared utilities: `run_command`, `resolve_executable`, `render_command`, `configured_bool`.

### Registry Client (`registry/client.py`)

`publish_to_registry()` assembles the registry API call:

1. Builds the metadata JSON from `DeliveryPayload`.
2. Encodes the metadata JSON part and the `.tar.zst` bundle bytes as `multipart/form-data`.
3. POSTs to `POST /skills/{slug}` with `Authorization: Bearer <token>`.
4. Returns a `RegistryPublishResult` (status code, body, request ID).

---

## Data Flow

### `inspect` command

```
1. CLI parses args → creates PublishContext
2. PublisherPipeline.run(context)
     discovery → discovery_gate
     identity  → identity_gate
     metadata  → metadata_gate
     security  → security_gate
     validation → validation_gate
     performance_exam (no gate)
     ranking (no gate)
     delivery (no gate)
     compression (no gate)
3. CLI prints evaluation report
4. Exit code: 0 if publish_decision != "block", else 1
```

### `publish` command

```
1. CLI parses args → creates PublishContext
2. PublisherPipeline.run(context) (same as inspect)
3. CLI prints evaluation report
4. If publish_decision == "block" → exit 1 (no upload)
5. build_bundle_bytes(context) → .tar.zst bytes (skill folder)
6. publish_to_registry(...)
     → POST /skills/{slug}  multipart/form-data
         metadata: JSON (intent, version, metadata, governance, relationships)
         bundle:   .tar.zst bytes
7. CLI prints registry result (status, request ID, response body)
8. Exit code: 0 on 2xx, else 1
```

### Gate failure handling

When a gate fails, `pipeline.run()` breaks out of the stage loop. Any stages after the failing gate do not run. The report still shows all stages that completed and the gate that halted execution.

---

## Core Invariants

1. **Governance inputs are caller-provided.** `trust_tier`, `namespace`, `artifact_origin`, `policy_pack_slug`, and `publisher_identity` come exclusively from CLI flags. They are never inferred from skill content.
2. **Security is blocking.** If garak is configured but does not produce a scored result, publishing is blocked. There is no local fallback for security.
3. **Upskill is advisory.** If Upskill is unavailable or unconfigured, performance metrics are recorded as absent. This does not block publish, but the performance score contributes 0.0 to the ranking.
4. **Stages are sequentially ordered.** No stage runs in parallel. Gate failures halt the pipeline at that point.
5. **The `.tar.zst` bundle is the authoritative skill artifact.** The compression stage produces the bundle that the registry stores. The delivery payload JSON is metadata only.
