# Aptitude - System Architecture

This document explains the cross-component design. Component-level details live
in the publisher, registry, and resolver docs.

## Design Principle

Aptitude is built around a strict ownership boundary:

| Component | Owns | Does not own |
| --- | --- | --- |
| Publisher | Local skill evaluation, payload assembly, bundle creation, registry upload | Registry authorization, lifecycle decisions, dependency solving |
| Registry | Immutable facts, governance state, discovery indexes, exact reads, catalog feeds, audit | Prompt interpretation, final selection, transitive solving, lock generation |
| Resolver | Intent handling, candidate reranking, final selection, solving, local policy, locks, materialization | Publication, authoritative metadata storage, registry governance |

## System Context

```mermaid
flowchart TD
    Author["Skill author / CI"]
    Developer["Developer / agent host"]
    Browser["Catalog visitor"]

    subgraph Local["Local tools"]
        Publisher["aptitude-publisher"]
        Resolver["aptitude / aptitude mcp"]
    end

    subgraph Hosted["Hosted services"]
        Website["Next.js website\naptitude-registry.dev"]
        Registry["Aptitude Registry API\napi.aptitude-registry.dev"]
        DB[("PostgreSQL\nNeon")]
    end

    Author --> Publisher
    Publisher -->|"POST /skills/{slug}\nmetadata + skill.tar.zst"| Registry

    Developer --> Resolver
    Resolver -->|"POST /discovery\nGET /skills\nGET /resolution\nGET content"| Registry

    Browser --> Website
    Website -->|"server-side catalog reads"| Registry

    Registry --> DB
```

## Main Data Flows

### Publish

```mermaid
sequenceDiagram
    participant Author as Author / CI
    participant Pub as Publisher
    participant Garak as garak
    participant Upskill as Upskill
    participant Reg as Registry
    participant DB as PostgreSQL

    Author->>Pub: aptitude-publisher publish /skill
    Pub->>Pub: discovery, identity, metadata
    Pub->>Garak: security scan
    Garak-->>Pub: scored findings or blocking failure
    Pub->>Pub: Anthropic validation
    Pub->>Upskill: optional performance evaluation
    Upskill-->>Pub: score or unavailable
    Pub->>Pub: ranking, delivery payload, bundle
    Pub->>Reg: POST /skills/{slug}
    Reg->>Reg: auth, schema validation, governance
    Reg->>DB: store version, content, metadata, selectors, search rows, audit
    Reg-->>Pub: registry response
```

### Discovery and Resolution

```mermaid
sequenceDiagram
    participant User as User / Agent
    participant Res as Resolver
    participant Reg as Registry
    participant DB as PostgreSQL

    User->>Res: aptitude install "query"
    Res->>Res: parse intent and build search request
    Res->>Reg: POST /discovery
    Reg->>DB: lexical search, semantic search, co-usage boost, governance filter
    DB-->>Reg: candidate rows
    Reg-->>Res: ordered candidate slugs
    Res->>Reg: GET /skills/{slug} and exact versions
    Res->>Res: candidate policy filter, rerank, select root
    Res->>Reg: GET /resolution/{slug}/{version}
    Res->>Res: solve graph and run graph governance
    Res->>Res: write aptitude.lock.json
    Res->>Reg: GET /skills/{slug}/{version}/content
    Res->>Res: verify checksum, extract, promote
```

### Lock Replay

```mermaid
flowchart LR
    Lock["aptitude.lock.json"] --> Parse["Parse lock"]
    Parse --> Plan["Build execution plan\nfrom install_order"]
    Plan --> Fetch["Fetch exact artifacts"]
    Fetch --> Verify["Verify compressed-byte checksums"]
    Verify --> Extract["Safe extract into staging"]
    Extract --> Promote["Promote target files"]
```

Lock replay skips discovery, ranking, and dependency solving. The lockfile is
the execution source of truth.

## Storage Model

The registry is the only component with a database. PostgreSQL stores:

- Skills and immutable versions.
- Opaque `.tar.zst` bundle bytes and SHA-256 content digests.
- Canonical version checksums.
- Metadata, tags, schemas, token/content estimates, and provenance.
- Direct authored dependency selectors.
- Search projections for lexical, semantic, and co-usage discovery.
- Governance state: namespace, trust tier, lifecycle, review state, promotion
  channel, ownership, policy packs, and trust evidence.
- Audit events for mutating operations and discovery/search activity.

Resolver lockfiles and execution plans are local outputs. Publisher artifacts
are local audit traces under `.publisher_artifacts/` in the skill folder.

## API Surface By Consumer

| Consumer | Registry routes |
| --- | --- |
| Publisher | `POST /skills/{slug}` |
| Resolver | `POST /discovery`, `GET /skills/{slug}`, `GET /skills/{slug}/{version}`, `GET /resolution/{slug}/{version}`, `GET /skills/{slug}/{version}/content` |
| Website | `GET /catalog/skills`, `GET /catalog/top-skills`, `GET /catalog/skill-graph`, `POST /catalog/search` |
| Telemetry bridge | `POST /catalog/star-events`, `GET /catalog/user-stars` |
| Operators | `GET /healthz`, `GET /readyz` |
| Admin/review | organization, namespace, ownership, governance, policy-pack, lifecycle, and trust-evidence endpoints |

All protected registry calls use service tokens of the form:

```text
Authorization: Bearer <token_id>.<token_secret>
```

## Operations Model

- Registry runs on Render and stores production data in Neon PostgreSQL.
- Migrations are Alembic-driven and use `MIGRATION_DATABASE_URL` when present.
- Semantic embeddings are indexed asynchronously by operator/workflow scripts.
- Registry observability uses structured logs plus OpenTelemetry export to
  Grafana Cloud when enabled.
- Website calls the registry server-side; browsers do not call registry APIs
  directly.

## Design Files

- [High-Level Design](high-level-design.md)
- [Publisher Architecture](publisher/architecture.md)
- [Registry Architecture](registry/architecture.md)
- [Resolver Architecture](resolver/architecture.md)
