# Aptitude Publisher - Architecture

The publisher is a single local Python process. Each run creates a
`PublishContext`, executes a fixed stage pipeline, records artifacts, and either
prints the inspection result or uploads a registry payload.

## Package Map

```text
publisher/
  publisher/
    app/
      cli.py              # argparse CLI: inspect and publish
      pipeline.py         # stage ordering and gate dispatch
    artifacts/
      bundle.py           # deterministic skill.tar.zst bundle builder
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
      compression.py
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
    DEL --> C["compression"]
```

Stages run sequentially. Gates run after `discovery`, `identity`, `metadata`,
`security`, and `validation`. A failed gate stops later stages.

## Stage Responsibilities

| Stage | Responsibility |
| --- | --- |
| `discovery` | Resolve the skill root, find `SKILL.md`, inventory files, parse frontmatter/body, collect git provenance. |
| `identity` | Resolve slug, version, and intent from CLI overrides or frontmatter. |
| `metadata` | Extract name, description, tags, schemas, maturity/security scores, token estimate, and optional repository signals. |
| `security` | Run garak and normalize findings, score, severity counts, and decision. |
| `validation` | Check skill folder and `SKILL.md` rules. |
| `performance_exam` | Run Upskill when configured; absent scores are advisory. |
| `ranking` | Combine upstream signals into weighted scores and a `publish_decision`. |
| `delivery` | Build the registry metadata/governance/relationship payload. |
| `compression` | Record compressed delivery-package artifacts for the local audit trail. |

The uploaded skill bundle is built separately by `artifacts/bundle.py` from the
raw skill folder, excluding `.publisher_artifacts/`.

## Gates

| Gate | Blocks on |
| --- | --- |
| `DiscoveryGate` | Missing skill root, missing `SKILL.md`, invalid parsed content, missing inventory artifact. |
| `IdentityGate` | Missing slug, version, or intent. |
| `MetadataGate` | Missing name, description, tags, `inputs_schema`, or `outputs_schema`; invalid maturity/security score range. |
| `SecurityGate` | garak did not run, did not score, produced an invalid decision, or returned `block`. |
| `ValidationGate` | No validation artifact, no checks, or any validation error. |

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

## Artifact Model

Two artifact streams exist:

- `.publisher_artifacts/` - local audit files from stages and gates.
- `skill.tar.zst` - uploaded deterministic bundle built from the skill folder.

Do not conflate them. The compression stage is part of the local trace; the
registry stores the uploaded skill bundle.

## Exit and Upload Behavior

The CLI blocks upload only when `publish_decision == "block"`. A
`review_required` decision is printed but is not currently an upload stop in
the CLI implementation. Registry-side governance may still reject the request.
