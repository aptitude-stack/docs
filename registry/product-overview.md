# Aptitude Registry — Product Overview

> Canonical product description for the Aptitude Registry service.

## What It Is

Aptitude Registry is the authoritative skill catalog backend in the Aptitude ecosystem. It is a control-first service: every piece of data it manages is immutable once written, digest-addressed, and governed by an explicit policy model before it is visible to callers.

Publishers (CI pipelines, human authors, automated agents) push versioned skill bundles to the registry. Resolvers, MCP clients, and CLI tools query the registry to discover candidate skills, read exact metadata, fetch immutable bundle artifacts, and retrieve direct dependency selectors. The registry never interprets intent, never selects a final skill set, and never builds execution plans — those responsibilities belong entirely to the caller.

## What the Registry Owns

- **Immutable publication.** Each `slug@version` is written once as a digest-addressed `.tar.zst` bundle. Artifact bytes are never patched after publication; a new version must be published if content changes.
- **Discovery candidate generation.** The `POST /discovery` endpoint returns an ordered list of candidate slugs using a hybrid lexical + semantic + co-usage ranking pipeline. It does not perform final selection.
- **Exact dependency reads.** `GET /resolution/{slug}/{version}` returns the direct authored `depends_on` selectors for one exact coordinate. Transitive graph resolution is a resolver concern.
- **Exact metadata and content fetch.** Callers can retrieve immutable structured metadata or the raw bundle artifact for any coordinate they already know.
- **Lifecycle governance.** Skills move through `published → deprecated → archived` under policy-enforced transitions. Only callers with the `admin` scope may trigger a transition.
- **Enterprise control plane.** Namespaces, organizations, policy packs, review states, promotion channels, and trust-tier classification are managed via admin-gated endpoints.
- **Audit.** Every publish, lifecycle transition, search, and governance change is recorded in a structured audit log.
- **Operational telemetry.** Health probes, readiness checks, structured logs, and OTLP metrics are exported to Grafana Cloud when `OTEL_ENABLED=true`.

## What the Registry Does Not Own

| Concern | Owned by |
| --- | --- |
| Prompt interpretation | Resolver / client |
| Re-ranking search results | Resolver / client |
| Final skill selection | Resolver / client |
| Transitive dependency solving | Resolver / client |
| Lock file generation | Resolver / client |
| Execution planning | Resolver / client |
| Runtime execution | Agent runtime |

This boundary is strict and intentional. The registry is a single source of truth for *what exists and what was declared*; everything downstream is the caller's responsibility.

## Technology Stack

| Component | Choice |
| --- | --- |
| Language | Python 3.12+ |
| Web framework | FastAPI |
| Validation | Pydantic |
| ORM / query | SQLAlchemy |
| Migrations | Alembic |
| Database | PostgreSQL (sole authoritative store) |
| Vector search | pgvector (`halfvec`, HNSW index) |
| Semantic embeddings | OpenAI Embeddings API |
| Bundle format | `.tar.zst` (`application/zstd`) |
| Package manager | uv |
| Linting / formatting | Ruff |
| Testing | pytest |
| Observability | Prometheus → Grafana Cloud (OTLP/HTTP) + Loki |
| Container | Docker / Docker Compose |

## Quick Start

Requirements: Python 3.12+, `uv`, Docker.

```bash
# Install dependencies
uv sync --extra dev

# Start the local stack with demo data (APP_ENV=dev)
make run-dev

# Run integration tests (uses a separate DB on port 5433)
make test
```

Local URLs once running:

- API: `http://127.0.0.1:8000`
- Swagger UI: `http://127.0.0.1:8000/docs`
- Grafana dashboard: `http://127.0.0.1:3000`

For production-like startup (no demo seed):

```bash
make run-prod
```

Teardown:

```bash
docker compose down -v
```

## Route Surface at a Glance

| Category | Key endpoints |
| --- | --- |
| Health | `GET /healthz`, `GET /readyz` |
| Publish | `POST /skills/{slug}` |
| Discovery | `POST /discovery` |
| Resolution | `GET /resolution/{slug}/{version}` |
| Exact fetch | `GET /skills/{slug}/{version}`, `GET /skills/{slug}/{version}/content` |
| Identity listing | `GET /skills/{slug}` |
| Lifecycle | `PATCH /skills/{slug}/{version}/status` |
| Catalog (website) | `GET /catalog/top-skills`, `GET /catalog/skill-graph`, `POST /catalog/search` |
| Stars | `POST /catalog/star-events`, `GET /catalog/user-stars` |
| Admin | `/admin/organizations`, `/admin/namespaces`, `/admin/policy-packs/{slug}`, etc. |

Full contract details live in [`reference/api-contract.md`](reference/api-contract.md).

## Key Reading Order

1. This document — product scope and ownership boundary.
2. [`architecture.md`](architecture.md) — system design and layer breakdown.
3. [`discovery-mechanism.md`](discovery-mechanism.md) — how hybrid search works.
4. [`tables.md`](tables.md) — PostgreSQL schema reference.
5. [`policies.md`](policies.md) — governance model and policy evaluation.
6. [`reference/api-contract.md`](reference/api-contract.md) — canonical HTTP contract.
