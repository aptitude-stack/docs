# Aptitude Registry — Database Tables

> Reference for all PostgreSQL tables, columns, constraints, and indexes.
> Schema is managed by Alembic; migration scripts in `alembic/versions/` are the canonical source.

## Table Index

| Table | Purpose |
| --- | --- |
| [`skills`](#skills) | Skill identity registry — one row per slug |
| [`skill_versions`](#skill_versions) | Immutable version records |
| [`skill_contents`](#skill_contents) | Immutable `.tar.zst` bundle artifacts |
| [`skill_metadata`](#skill_metadata) | Normalized queryable metadata |
| [`skill_relationship_selectors`](#skill_relationship_selectors) | Authored relationship declarations |
| [`skill_search_documents`](#skill_search_documents) | Lexical search projection |
| [`skill_search_embeddings`](#skill_search_embeddings) | Semantic embedding vectors (pgvector) |
| [`skill_co_usage_pairs`](#skill_co_usage_pairs) | Pre-computed co-usage statistics |
| [`skill_usage_observation_runs`](#skill_usage_observation_runs) | Source snapshots for co-usage computation |
| [`skill_usage_observations`](#skill_usage_observations) | Per-run skill observation events |
| [`skill_user_stars`](#skill_user_stars) | Per-user starred skill relations |
| [`organizations`](#organizations) | Enterprise organization records |
| [`namespaces`](#namespaces) | Namespace ownership boundaries |
| [`policy_packs`](#policy_packs) | Registry-enforced policy pack rules |
| [`trust_evidence`](#trust_evidence) | Append-only trust attestation records |
| [`audit_events`](#audit_events) | Structured audit log |

---

## skills

One row per skill slug. Mutable aggregate counters live here; content and versions live in child tables.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | Internal surrogate key |
| `slug` | `text` | not null | — | Globally unique skill identifier, e.g. `python-lint` |
| `namespace_fk` | `bigint` | not null | — | FK → `namespaces.id` (RESTRICT on delete) |
| `install_count` | `bigint` | not null | `0` | Mutable aggregate; incremented by install telemetry |
| `star_count` | `bigint` | not null | `0` | Mutable aggregate; updated by `POST /catalog/star-events` |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |
| `updated_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Constraints:** `uq_skills_slug` (unique on `slug`).

**Indexes:** `ix_skills_namespace_fk`.

---

## skill_versions

One row per immutable `slug@version` coordinate. Immutable once written; only mutable enterprise workflow columns (`artifact_origin`, `review_state`, `promotion_channel`, `policy_pack_fk`) can be updated post-publish via governance endpoints.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `skill_fk` | `bigint` | not null | — | FK → `skills.id` (CASCADE) |
| `version` | `text` | not null | — | Semver string, e.g. `1.2.3` |
| `content_fk` | `bigint` | not null | — | FK → `skill_contents.id` (RESTRICT) |
| `metadata_fk` | `bigint` | not null | — | FK → `skill_metadata.id` (RESTRICT) |
| `checksum_digest` | `varchar(64)` | not null | — | SHA-256 of canonical version payload JSON |
| `lifecycle_status` | `text` | not null | `published` | `published`, `deprecated`, or `archived` |
| `lifecycle_changed_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | Timestamp of last lifecycle transition |
| `trust_tier` | `text` | not null | `untrusted` | `untrusted`, `internal`, or `verified` |
| `artifact_origin` | `text` | not null | `internal` | `internal`, `imported`, `verified`, or `restricted` |
| `review_state` | `text` | not null | `approved` | `pending_review`, `approved`, or `rejected` |
| `promotion_channel` | `text` | not null | `prod` | `dev`, `staging`, or `prod` |
| `policy_pack_fk` | `bigint` | null | — | FK → `policy_packs.id` (SET NULL) |
| `provenance_repo_url` | `text` | null | — | Publish-time provenance |
| `provenance_commit_sha` | `text` | null | — | Hex commit SHA (7–64 chars) |
| `provenance_tree_path` | `text` | null | — | In-repo path for the skill source |
| `provenance_publisher_identity` | `text` | null | — | Publisher token identity |
| `policy_profile_at_publish` | `text` | null | — | Active policy profile name at publish time |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |
| `published_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Constraints:** `uq_skill_versions_skill_fk_version` (unique on `skill_fk, version`); check constraints for `lifecycle_status`, `trust_tier`, `artifact_origin`, `review_state`, `promotion_channel`.

**Indexes:** `ix_skill_versions_skill_fk`, `ix_skill_versions_content_fk`, `ix_skill_versions_metadata_fk`, `ix_skill_versions_skill_fk_published_at_id`, `ix_skill_versions_skill_fk_version`, `ix_skill_versions_policy_pack_fk`.

---

## skill_contents

Immutable artifact storage. Each row holds the raw `.tar.zst` bytes for one content digest. Multiple versions may reference the same content row if their bundles are byte-identical.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `payload` | `bytea` | not null | — | Raw `.tar.zst` bundle bytes |
| `media_type` | `text` | not null | — | Always `application/zstd` |
| `storage_size_bytes` | `bigint` | not null | — | Byte length of `payload` |
| `checksum_digest` | `varchar(64)` | not null | unique | SHA-256 of `payload` |

**Constraints:** unique on `checksum_digest`.

---

## skill_metadata

Normalized queryable metadata attached to one version. Stored separately from content to allow metadata queries without loading bundle bytes.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `name` | `text` | not null | — | Human-readable skill name |
| `description` | `text` | null | — | |
| `tags` | `text[]` | not null | `'{}'` | Raw tag list |
| `inputs_schema` | `jsonb` | null | — | JSON Schema for skill inputs |
| `outputs_schema` | `jsonb` | null | — | JSON Schema for skill outputs |
| `token_estimate` | `integer` | null | — | Estimated token cost |
| `maturity_score` | `float` | null | — | 0–1 maturity signal |
| `security_score` | `float` | null | — | 0–1 security signal |

---

## skill_relationship_selectors

Authored relationship declarations for one version. Stored as ordered rows with an `edge_type` and an `ordinal` to preserve declaration order.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `source_skill_version_fk` | `bigint` | not null | — | FK → `skill_versions.id` (CASCADE) |
| `edge_type` | `text` | not null | — | `depends_on`, `extends`, `conflicts_with`, or `overlaps_with` |
| `ordinal` | `integer` | not null | — | Declaration order within one `edge_type` |
| `target_slug` | `text` | not null | — | Target skill slug |
| `target_version` | `text` | null | — | Exact pinned version (if any) |
| `version_constraint` | `text` | null | — | Semver constraint string (if any) |
| `optional` | `boolean` | null | — | Whether the dependency is optional |
| `markers` | `text[]` | not null | — | Environment markers |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Constraints:** check constraint on `edge_type`.

**Indexes:** `ix_skill_relationship_selectors_source_edge_type_ordinal` on `(source_skill_version_fk, edge_type, ordinal)`.

---

## skill_search_documents

Denormalized search projection, one row per version. Maintained in sync with `skill_versions` at publish time and updated when governance state changes. This is the primary table for lexical discovery queries.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `skill_version_fk` | `bigint` | not null | PK | FK → `skill_versions.id` (CASCADE) |
| `slug` | `text` | not null | — | |
| `normalized_slug` | `text` | not null | — | Lowercased, punctuation-stripped |
| `version` | `text` | not null | — | |
| `name` | `text` | not null | — | |
| `normalized_name` | `text` | not null | — | |
| `description` | `text` | null | — | |
| `tags` | `text[]` | not null | — | Raw tags |
| `normalized_tags` | `text[]` | not null | — | Lowercased tags |
| `lifecycle_status` | `text` | not null | `published` | |
| `trust_tier` | `text` | not null | `untrusted` | |
| `namespace` | `text` | not null | `public` | |
| `artifact_origin` | `text` | not null | `internal` | |
| `review_state` | `text` | not null | `approved` | |
| `promotion_channel` | `text` | not null | `prod` | |
| `policy_pack_slug` | `text` | null | — | |
| `search_vector` | `tsvector` | not null | `''::tsvector` | Pre-computed full-text search vector |
| `published_at` | `timestamptz` | not null | — | |
| `content_size_bytes` | `bigint` | not null | — | |
| `usage_count` | `bigint` | not null | `0` | |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Indexes:** B-tree on `normalized_slug`, `normalized_name`, `published_at`, `content_size_bytes`, `lifecycle_status`, `trust_tier`, `namespace`, `review_state`, `promotion_channel`; GIN on `normalized_tags` and `search_vector`.

---

## skill_search_embeddings

Semantic embedding vectors for one version at a given model/dimension pair. Managed by the background embedding indexer.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `skill_version_fk` | `bigint` | not null | PK component | FK → `skill_versions.id` (CASCADE) |
| `embedding_model` | `text` | not null | PK component | e.g. `text-embedding-3-small` |
| `embedding_dimensions` | `integer` | not null | — | Must be `1536` (check constraint) |
| `source_checksum_digest` | `text` | not null | — | SHA-256 of the text used to generate the vector |
| `embedding_vector` | `halfvec(1536)` | null | — | pgvector half-precision float vector |
| `index_status` | `text` | not null | `pending` | `pending`, `indexed`, `failed`, or `stale` |
| `indexed_at` | `timestamptz` | null | — | Timestamp when status last became `indexed` |
| `last_error` | `text` | null | — | Last error message if `failed` |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |
| `updated_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Primary key:** `(skill_version_fk, embedding_model)`.

**Indexes:** B-tree on `(embedding_model, index_status)`; B-tree on `source_checksum_digest`; HNSW on `embedding_vector halfvec_cosine_ops` (`m=16`, `ef_construction=64`) — partial index where `embedding_vector IS NOT NULL AND index_status = 'indexed'`.

---

## skill_co_usage_pairs

Pre-computed co-usage statistics for pairs of skill identities, derived from `skill_usage_observations`. Updated by an offline aggregation job.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `anchor_skill_fk` | `bigint` | not null | PK component | FK → `skills.id` (CASCADE) |
| `related_skill_fk` | `bigint` | not null | PK component | FK → `skills.id` (CASCADE) |
| `observation_count` | `bigint` | not null | `0` | Co-observed install events |
| `distinct_run_count` | `bigint` | not null | `0` | Sessions where both were seen |
| `co_usage_rate` | `numeric(10,6)` | not null | `0` | `distinct_run_count / total_runs_for_anchor` |
| `lift_score` | `numeric(10,6)` | not null | `0` | Normalized lift |
| `pmi_score` | `numeric(10,6)` | not null | `0` | Pointwise mutual information |
| `last_observed_at` | `timestamptz` | null | — | |
| `window_days` | `integer` | not null | — | Observation window for this row |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |
| `updated_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Constraints:** `anchor_skill_fk <> related_skill_fk`; counts non-negative; `window_days > 0`.

**Indexes:** B-tree on `related_skill_fk`.

---

## skill_usage_observation_runs

One row per source snapshot ingested for co-usage computation. Idempotent on `(source, source_digest)`.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | generated PK | |
| `source` | `text` | not null | — | Logical source identifier |
| `source_digest` | `text` | not null | — | SHA-256 of the source snapshot |
| `observed_at` | `timestamptz` | not null | — | When observations were made |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Constraints:** unique on `(source, source_digest)`.

---

## skill_usage_observations

Individual skill observations within one run. One row per `(run, skill)` pair.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | generated PK | |
| `run_fk` | `bigint` | not null | — | FK → `skill_usage_observation_runs.id` (CASCADE) |
| `skill_fk` | `bigint` | not null | — | FK → `skills.id` (CASCADE) |
| `skill_slug` | `text` | not null | — | Denormalized slug for query convenience |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Constraints:** unique on `(run_fk, skill_fk)`.

**Indexes:** B-tree on `skill_fk`.

---

## skill_user_stars

Per-user starred skill relations. Managed by `POST /catalog/star-events` with a `user_subject`. `skills.star_count` is kept in sync via triggers/application logic.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `user_subject` | `text` | not null | — | Trusted caller-supplied user identifier |
| `skill_fk` | `bigint` | not null | — | FK → `skills.id` (CASCADE) |
| `created_at` | `timestamptz` | not null | `now()` | |

**Constraints:** unique on `(user_subject, skill_fk)`.

**Indexes:** B-tree on `user_subject`; B-tree on `skill_fk`.

---

## organizations

Enterprise organization records. One organization can own multiple namespaces.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `slug` | `text` | not null | unique | e.g. `acme-corp` |
| `display_name` | `text` | not null | — | |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |
| `updated_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

The built-in `public` organization and its `public` namespace are seeded at migration time.

---

## namespaces

Namespace ownership boundaries. Skills are assigned to exactly one namespace; namespace visibility controls whether versions are discoverable without namespace grants.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `organization_fk` | `bigint` | not null | — | FK → `organizations.id` (CASCADE) |
| `slug` | `text` | not null | unique | e.g. `acme-internal` |
| `visibility` | `text` | not null | — | `public` or `private` |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |
| `updated_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Constraints:** `visibility IN ('public', 'private')`.

**Indexes:** B-tree on `organization_fk`.

---

## policy_packs

Registry-enforced policy-pack rule sets. A version may reference at most one policy pack; the pack's `rules` JSON controls visibility and publish eligibility beyond the default governance model.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `slug` | `text` | not null | unique | |
| `description` | `text` | null | — | |
| `rules` | `jsonb` | not null | — | Policy rule document |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |
| `updated_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

See [`policies.md`](policies.md) for the supported `rules` keys.

---

## trust_evidence

Append-only trust attestation records for one version. Evidence is never deleted or modified; the registry exposes type, subject, digest, and URI — not the raw payload.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `bigint` | not null | autoincrement PK | |
| `skill_version_fk` | `bigint` | not null | — | FK → `skill_versions.id` (CASCADE) |
| `evidence_type` | `text` | not null | — | e.g. `sbom`, `signed_release`, `vulnerability_scan` |
| `subject` | `text` | not null | — | What the evidence covers |
| `digest` | `text` | null | — | Optional content digest of the evidence artifact |
| `uri` | `text` | null | — | Optional link to the evidence artifact |
| `payload` | `jsonb` | null | — | Optional structured evidence payload (not exposed via API) |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

**Indexes:** B-tree on `skill_version_fk`.

---

## audit_events

Structured audit log for all registry operations. Append-only; rows are never updated or deleted.

| Column | Type | Nullable | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | `integer` | not null | autoincrement PK | |
| `event_type` | `varchar(100)` | not null | — | e.g. `skill.published`, `status.transitioned`, `search.performed` |
| `payload` | `json` | null | — | Event-specific structured payload |
| `created_at` | `timestamptz` | not null | `CURRENT_TIMESTAMP` | |

Search audit payloads are intentionally redacted — they record query *shape* (term present/absent, tag count, result count) without storing raw query text.
