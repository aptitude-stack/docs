# Aptitude Publisher - Architecture

The publisher is a single local Python process. Each run creates a
`PublishContext`, executes a fixed stage pipeline, checkpoints one cache report,
and either prints the inspection result or uploads a registry payload.

## Package Map

```text
publisher/
  publisher/
    app/
      cli.py              # argparse CLI: inspect and publish
      pipeline.py         # stage ordering and gate dispatch
    artifacts/
      bundle.py           # deterministic skill.tar.zst bundle builder
      report.py           # one latest JSON report outside the source tree
    domain/
      models.py           # PublishContext and stage/gate data models
    gates/
      discovery.py
      identity.py
      metadata.py
      security.py
      validation.py
    integrations/
      external_tools.py
      garak_security.py
      github_api.py
      upskill_eval.py
    registry/
      client.py           # multipart POST /skills/{slug}
    stages/
      discovery.py
      identity.py
      metadata.py
      security.py
      validation.py
      performance_exam.py
      ranking.py
      delivery.py
```

## Pipeline

```mermaid
flowchart LR
    D["discovery"] --> DG{"gate"}
    DG --> I["identity"] --> IG{"gate"}
    IG --> M["metadata"] --> MG{"gate"}
    MG --> S["security"] --> SG{"gate"}
    SG --> V["validation"] --> VG{"gate"}
    VG --> P["performance_exam"]
    P --> R["ranking"]
    R --> DEL["delivery"]
    DEL --> OUT["latest cache report"]
```

Stages run sequentially. Gates run after `discovery`, `identity`, `metadata`,
`security`, `validation`, and `performance_exam`. A failed gate stops later
stages; the performance gate records evaluator failure and continues to
ranking so the final decision can include that evidence.

## Stage Responsibilities

| Stage | Responsibility |
| --- | --- |
| `discovery` | Resolve the skill root, find `SKILL.md` and `aptitude.yaml`, inventory files, parse source metadata, and collect git provenance. |
| `identity` | Resolve slug from `SKILL.md` `name`; resolve version and intent from CLI overrides or `aptitude.yaml`. |
| `metadata` | Extract name, description, license, and compatibility from `SKILL.md`; extract tags, schemas, numeric hints, and relationships from `aptitude.yaml`. |
| `security` | Run garak and normalize findings, score, severity counts, and decision. |
| `validation` | Check skill folder and `SKILL.md` rules. |
| `performance_exam` | Run Upskill and normalize scored or inconclusive evidence. |
| `ranking` | Combine upstream signals into weighted scores and a `publish_decision`. |
| `delivery` | Build the registry metadata/governance/relationship payload. |

The uploaded skill bundle is built separately by `artifacts/bundle.py` from the
raw skill folder. It includes `aptitude.yaml` and preserves
`agents/openai.yaml` when present, while excluding the preserved
`.publisher_artifacts/` directory.

## Gates

| Gate | Blocks on |
| --- | --- |
| `DiscoveryGate` | Missing skill root, missing `SKILL.md` or `aptitude.yaml`, invalid parsed content, or incomplete inventory. |
| `IdentityGate` | Missing slug, version, or intent. |
| `MetadataGate` | Missing name, description, tags, `inputs_schema`, or `outputs_schema`; invalid maturity/security score range. |
| `SecurityGate` | garak did not run, did not score, produced an invalid decision, or returned `block`. |
| `ValidationGate` | No validation result, no checks, or any validation error. |

## Upload Contract

`publish` sends a multipart request:

```text
POST /skills/{slug}
Authorization: Bearer <token>

metadata: application/json
bundle: application/zstd, filename skill.tar.zst
```

`registry/client.py` builds the metadata JSON from the delivery payload and
encodes the bundle bytes as multipart form data.

## Report and Bundle Model

The publisher retains one latest JSON report per canonical skill directory. Its
path is `<cache-root>/aptitude/publisher/<sha256(canonical-absolute-skill-directory)>.json`,
where the cache root is the absolute `XDG_CACHE_HOME` when configured or
`~/.cache` otherwise. The report is atomically replaced and contains the
`schema_version`, `skill_root`, `updated_at`, `status`, `stages`, `gates`,
`evidence`, `warnings`, `error`, and nested `inspection_receipt` fields.

Stage and gate records, normalized security and performance evidence, exact
evaluation decisions, and the signed inspection receipt remain in that report.
Raw evaluator transcripts, credentials, environment dumps, and temporary paths
are redacted or omitted. Evaluators use temporary copies and working directories
outside the source tree and clean them after success, failure, or timeout.

The uploaded `skill.tar.zst` bundle is separate from the report. Existing
`.publisher_artifacts/` directories are preserved as historical content,
excluded from inventory and bundles, and never read or written by the current
publisher.

MCP responses expose the report location as `report_path`; `artifacts_dir` is no
longer part of the response contract.

## Exit and Upload Behavior

The CLI blocks upload only when `publish_decision == "block"`. A
`review_required` decision is printed but is not currently an upload stop in
the CLI implementation. Registry-side governance may still reject the request.
