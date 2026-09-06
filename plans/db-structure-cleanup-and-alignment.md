---
goal: Deep database structure cleanup and application alignment
version: "1.0"
date_created: "2026-09-06"
last_updated: "2026-09-06"
owner: Aptitude
status: In progress
tags: [architecture, database, migration, cleanup]
---

# Introduction

![Status: In progress](https://img.shields.io/badge/status-In_progress-yellow)

Simplify the registry’s PostgreSQL schema, remove redundant state, enforce data integrity, and align application code, tests, benchmarks, and documentation.

Save this plan as `/Users/yonatan/Dev/Aptitude/docs/plans/db-structure-cleanup-and-alignment.md` when execution is enabled. No file or database changes have been made in Plan mode.

The selected approach preserves governance, semantic search, and co-usage features; uses a coordinated maintenance window; and makes per-user star records authoritative.

## 1. Requirements & Constraints

Paths below are relative to `/Users/yonatan/Dev/Aptitude`.

- **REQ-001 — Preserve published data:** Preserve skill/version IDs, artifact bytes, both checksum values, metadata values, timestamps, authored selector order, governance state, trust evidence, user stars, and audit history. Never recompute historical version checksums.
- **REQ-002 — Preserve capabilities:** Retain organizations, namespaces, policy packs, trust evidence, content deduplication, semantic indexing, co-usage, and derived search/graph tables.
- **REQ-003 — Consolidate metadata:** Move `name`, `description`, `tags`, `token_estimate`, `maturity_score`, `security_score`, and `overall_score` into `skill_versions`, retaining their types, nullability, and defaults; remove `metadata_fk` and `skill_metadata`.
- **REQ-004 — Remove duplicated counters:** Remove physical `skills.star_count` and `skill_search_documents.usage_count`; derive stars from `skill_user_stars` and return `skills.install_count AS usage_count` in search queries.
- **REQ-005 — Simplify derived signals:** Remove `skill_usage_observations.skill_slug`, `skill_co_usage_pairs.observation_count`, `skill_co_usage_pairs.pmi_score`, and `skill_search_embeddings.embedding_dimensions`.
- **REQ-006 — Preserve public response shapes:** Keep nested `metadata`, numeric `star_count`, search ranking fields, graph responses, and embedding work-item interfaces. The intentional API change is that `/catalog/star-events` requires a nonblank `user_subject`; missing/null subjects return validation error `422`.
- **SEC-001 — Preserve identity boundaries:** The website supplies `user_subject` from its verified session, and registry star endpoints remain restricted to trusted telemetry callers. Do not introduce browser-supplied identity forwarding or a new authentication system.
- **CON-001 — Migration history:** Append revision `0013_db_structure_cleanup` after `0012_remove_metadata_schemas`; preserve historical migrations unchanged.
- **CON-002 — Maintenance cutover:** Old application processes and workers must not access the contracted schema. Pause public traffic, scheduled jobs, workflows, and other writers before migration.
- **CON-003 — Execution boundaries:** Local implementation and verification are separate from production approval. Do not commit, push, deploy, change credentials, or mutate hosted databases implicitly.
- **PAT-001 — Existing stack:** Use SQLAlchemy, Alembic, PostgreSQL/pgvector, `uv`, and existing tests. Add no database abstraction framework or dependency solely for this cleanup.
- **PAT-002 — Keep useful boundaries:** Retain tags/markers as arrays, existing identity keys, both version timestamps, and the distinction between artifact checksums and version checksums.

### Required database invariants

| Area | Final invariant |
|---|---|
| Version metadata | Nullable scores are within `[0,1]`; nullable token estimates are nonnegative. |
| Counters and sizes | `skills.install_count >= 0`; search content sizes are nonnegative; stored artifact size equals `octet_length(payload)`. |
| Selectors | Unique `(source_skill_version_fk, edge_type, ordinal)`; nonnegative ordinal; dependencies have exactly one exact version or range; other edges require an exact version and prohibit dependency-only hints. |
| Embeddings | Storage remains `halfvec(1536)`; indexed rows require a vector and `indexed_at`; configuration rejects dimensions other than 1536 before database/provider work. |
| Co-usage | Retain `distinct_run_count`, directional rates, lift, window, and timestamps; require nonnegative counts/lift, rates within `[0,1]`, positive windows, and distinct skills. |
| Graph source | Composite FK `(source_skill_version_fk, source_skill_fk)` references `(skill_versions.id, skill_versions.skill_fk)` when a source version exists. |
| Graph target | Composite FK `(target_skill_fk, target_slug)` references `(skills.id, skills.slug)` when a target ID exists; unresolved authored targets retain nullable IDs. |
| Graph shape | Authored edges require a source version and an authored edge type; co-usage edges require no source version, `relates_to`, and canonical ordered skill IDs. |

Replace superseded scalar graph FKs with the composite constraints while retaining the direct source-skill FK needed by co-usage rows. Add the necessary referenced composite unique keys; these indexes serve integrity enforcement.

## 2. Implementation Steps

### Implementation Phase 1

- **GOAL-001:** Establish a reproducible preflight and populated migration fixture.
- Tasks are sequential.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-001 | Create `registry/scripts/check_db_structure.py` with before/after checks for revision, required constraints, metadata coverage, redundant-value differences, and canonical-data fingerprints, verifying that it performs only reads and returns nonzero for blocking violations. | Completed | 2026-09-06 |
| TASK-002 | Add populated cleanup fixtures to `registry/tests/integration/test_migrations.py` at revision `0012`, covering multiple versions, shared content/metadata, nullable fields, governance, selectors, stars, indexed embeddings, and nonempty co-usage data. | Completed | 2026-09-06 |

Preflight must report orphan metadata and invalid canonical data as blockers rather than silently deleting or correcting them. Shared metadata is supported by copying its values into every referencing version.

Record historical star totals and their differences from user-row counts in the preflight report. Signed-in user rows determine final counts; do not manufacture users to preserve anonymous increments.

### Implementation Phase 2

- **GOAL-002:** Implement the final schema and all affected readers/writers.
- Depends on Phase 1; tasks are sequential because they share repository code.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-003 | Update `registry/app/persistence/models/skill_version.py`, `create_version()`, and detail mappers to store metadata on versions, remove persistence-only `SkillMetadata` imports/joins, and verify unchanged nested API metadata. | Completed | 2026-09-06 |
| TASK-004 | Replace the stored star counter in `registry/app/persistence/models/skill.py` with a read-only SQLAlchemy `column_property` counting user-star rows, and update `apply_user_star_events()` to return freshly queried counts after each flushed event. | Completed | 2026-09-06 |
| TASK-005 | Require `user_subject` in registry telemetry DTOs/routes, remove aggregate-only service/port/repository methods, and require the existing session subject in `website/src/lib/registry-client.ts:submitStarEvents()`, verified by authenticated and missing-subject tests. | Completed | 2026-09-06 |
| TASK-006 | Update lexical and semantic search SQL to join versions and skills for `install_count AS usage_count`, remove projection-counter writes from `record_install()` and `build_search_document()`, and verify newly published versions immediately receive the existing popularity signal. | Completed | 2026-09-06 |
| TASK-007 | Update observation imports, pair aggregation, and graph synchronization in `registry/app/persistence/skill_registry_repository.py` to remove obsolete columns and graph-evidence keys while preserving ranking, thresholds, directional pairs, and canonical graph pairs. | Completed | 2026-09-06 |
| TASK-008 | Remove persisted-dimension SQL from embedding creation, backfill, claiming, indexing, and search; retain the 1536 dimension in provider/work-item interfaces and validate it in settings, verified by successful indexing and early rejection of unsupported dimensions. | Completed | 2026-09-06 |
| TASK-009 | Make `SkillContent.payload` deferred with accidental lazy loading rejected, explicitly load it for `get_version_content()`, and verify catalog/detail/search reads omit payload bytes while downloads remain byte-identical. | Completed | 2026-09-06 |
| TASK-010 | Add the required checks, selector uniqueness, graph composite FKs, and graph partial unique indexes to ORM-owned models, removing the exact duplicate version index and replacing the selector ordering index with its unique equivalent. | Completed | 2026-09-06 |

Implementation details:

- Preserve existing ordered star-event responses and idempotency. Acquire skill locks in deterministic ID order for multi-skill star batches.
- Keep the user-star surrogate ID and current natural uniqueness constraint; changing those keys adds no necessary benefit here.
- Remove `observation_count` and `pmi_score` from existing co-usage graph JSON evidence as well as future writes. Preserve other evidence keys.
- Keep remaining search projection fields and their transactional governance synchronization.
- Preserve manually managed semantic/co-usage tables as Alembic-owned SQL tables; do not add ORM models or a pgvector Python dependency merely for metadata completeness.

### Implementation Phase 3

- **GOAL-003:** Provide a transactional populated-database migration and downgrade.
- Depends on Phase 2.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-011 | Implement `registry/alembic/versions/0013_db_structure_cleanup.py:upgrade()` to validate preconditions, copy metadata, verify copied values, enforce final constraints, remove obsolete structures, and verify an atomic upgrade on populated fixtures. | Completed | 2026-09-06 |
| TASK-012 | Implement the same migration’s `downgrade()` to reconstruct the `0012` table shape and derived columns without altering canonical published data, verified by upgrade–downgrade–upgrade tests. | Completed | 2026-09-06 |

Migration contract:

1. Use the direct migration connection and one transaction, with `lock_timeout='5s'` and `statement_timeout='120s'`.
2. Abort on invalid canonical rows, ambiguous selectors, orphan metadata, or inconsistent graph identifiers; do not guess corrective values.
3. Add version metadata columns, copy by `metadata_fk`, compare every copied field using null-safe equality, then enforce nullability and checks.
4. Drop the obsolete FK/index/table only after verification; use explicit drops rather than broad `CASCADE`.
5. Remove redundant signal/counter columns and obsolete graph-evidence keys.
6. Retain all artifact bytes, historical checksums, embedding vectors/statuses, identity IDs, and timestamps.
7. Drop `ix_skill_versions_skill_fk_version`; retain the unique constraint backing the same key.

Downgrade reconstructs one metadata row per version, synchronizes its sequence, restores star totals from user rows and search usage from install counts, restores observation slugs by FK, and reconstructs count/PMI/dimension fields. Metadata surrogate IDs and recomputed diagnostic values need not reproduce their former representation; canonical version data must match.

### Implementation Phase 4

- **GOAL-004:** Align tooling and documentation and prove the complete behavior.
- Depends on Phase 3.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-013 | Update `registry/scripts/benchmark_discovery_search.py`, current fixtures, Bruno requests, DTO examples, and schema assertions to use the final structure, verifying that benchmark cleanup remains limited to its synthetic records. | Completed | 2026-09-06 |
| TASK-014 | Update schema, API, governance, and deployment documentation under `registry/docs/` with final ownership, signed-in stars, retained projections, and the maintenance procedure, verifying that current instructions contain no retired-column references. | Completed | 2026-09-06 |
| TASK-015 | Run registry quality/full integration checks and website tests/typecheck, inspect SQL query shapes and representative query plans, and obtain a defect-focused review with all actionable findings resolved. | Completed | 2026-09-06 |

Historical migrations, historical changelogs, and tests explicitly exercising old revisions retain their old-schema references.

### Implementation Phase 5

- **GOAL-005:** Rehearse and perform an approved maintenance-window release.
- Depends on Phase 4 and separate authorization for hosted actions.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-016 | Rehearse revision `0013`, application smoke checks, and downgrade on a dedicated Neon child of the production branch, verifying canonical fingerprints and recording migration/rollback duration. | Pending approval | 2026-09-06 |
| TASK-017 | Execute the maintenance procedure documented in `registry/docs/architecture/render-neon-deployment.md` against the explicitly verified production target, marking completion only after schema, deployed revision, and public read checks all pass. | Pending approval | 2026-09-06 |

Maintenance sequence:

1. Verify project `bitter-night-16887852`, branch `production` / `br-calm-bonus-ambx0ki5`, database `aptitude`, current migration, and approved application commit.
2. Temporarily disable the automatic master-push deployment workflow **before merging the destructive migration**, recording its prior enabled state.
3. Enable Render maintenance mode for `aptitude-registry-api`; pause the semantic cron trigger and drain active workflow/indexing runs and other writers.
4. Verify active work has finished; maintenance mode alone does not stop private-network clients or background jobs.
5. Create a retained pre-cutover Neon backup branch and record its ID; run the final preflight after writes are paused.
6. Apply Alembic migration `0013` once through the direct URL and verify the new revision and canonical-data fingerprints.
7. Deploy the approved registry and worker revision while maintenance remains enabled; perform internal read-only smoke checks.
8. Reopen public traffic only after internal verification; verify public health, readiness, metadata, resolution, and search, then resume workers/schedules and restore the deployment workflow’s prior state.

Do not run the existing public-readiness polling step while Render maintenance mode intentionally returns `503`. Use a controlled release for this cutover rather than the normal migrate-before-deploy sequence.

On migration failure, preserve maintenance and rely on transaction rollback. On application verification failure, keep writers paused, run the rehearsed downgrade using the new release’s migration code, deploy the previous application revision, and verify before reopening. If downgrade fails, keep maintenance enabled and escalate recovery using the retained backup; never discard intervening data automatically.

## 3. Alternatives

- **ALT-001:** Staged expand/backfill/contract releases were rejected in favor of the selected maintenance window.
- **ALT-002:** Keeping metadata as a shared-primary-key child table was rejected because its fields belong to the version and have no independent lifecycle.
- **ALT-003:** Retaining star/search counter caches with additional synchronization was rejected because querying their authoritative sources removes the drift paths.
- **ALT-004:** Removing dormant enterprise/co-usage features was rejected; the user selected preservation and simplification.
- **ALT-005:** Rewriting historical migrations, replacing SQLAlchemy, and adding generic projection infrastructure are excluded.

## 4. Dependencies

- **DEP-001:** Existing SQLAlchemy/Alembic stack and PostgreSQL with pgvector; the inspected production server is PostgreSQL 17.11.
- **DEP-002:** Local disposable PostgreSQL test database from the existing Makefile.
- **DEP-003:** Website identity forwarding already uses a verified session subject; its required client type must remain aligned with registry validation.
- **DEP-004:** Approved Neon rehearsal/backup branches and Render maintenance, worker, and deployment access are required only for Phase 5.
- **DEP-005:** Production may change during implementation; recheck revision and preflight immediately before rehearsal and cutover.

## 5. Files

- **FILE-001:** `registry/app/persistence/models/`, repository/support modules, and `registry/alembic/env.py` — canonical fields, queries, loaders, and model registration.
- **FILE-002:** `registry/alembic/versions/0013_db_structure_cleanup.py` — new forward and reverse migration.
- **FILE-003:** `registry/app/core/`, telemetry DTO/routes/examples, and website registry client — counter and dimension interfaces.
- **FILE-004:** `registry/scripts/check_db_structure.py` and discovery benchmark — verification and synthetic-data alignment.
- **FILE-005:** Existing registry/website tests, Bruno requests, and registry reference/deployment documentation — behavior and operational contracts.

## 6. Testing

- **TEST-001:** Upgrade populated `0012` data and compare canonical fingerprints, including artifact bytes, metadata, checksums, selectors, governance, stars, and indexed embeddings.
- **TEST-002:** Test shared metadata/content, nullable metadata, invalid scores, duplicate selector positions, malformed selectors, and inconsistent graph identifiers; invalid canonical data must abort the migration atomically.
- **TEST-003:** Exercise empty-database upgrade, populated downgrade to `0012`, re-upgrade, and existing historical migration tests.
- **TEST-004:** Execute invalid direct SQL inserts to prove database constraints reject invalid states independently of API validation.
- **TEST-005:** Verify repeated/concurrent star and unstar requests, multi-skill batches, joined/aliased computed-count queries, and `422` for missing/null subjects.
- **TEST-006:** Publish a second version after installs exist and verify lexical and semantic ranking immediately use the same authoritative install count.
- **TEST-007:** Exercise embedding backfill, claim/reclaim, success/failure, fixed-dimension validation, observation deduplication, co-usage thresholds, graph activation, and decay.
- **TEST-008:** Capture emitted SQL to prove metadata/catalog/search reads avoid artifact payload and do not introduce per-result count queries; verify exact downloads remain unchanged.
- **TEST-009:** Run `make quality` and `make test` from `registry/`; run `bun run test` and `bun run typecheck` from `website/`.
- **TEST-010:** Never point the ordinary integration suite at production or the populated rehearsal clone: its fixture drops and recreates the public schema. Use migration/preflight and dedicated smoke checks on the clone.

## 7. Risks & Assumptions

- **RISK-001:** The existing production pipeline migrates before deploying; disabling that automatic path before the destructive release is mandatory.
- **RISK-002:** Missing worker coordination can let old code access removed columns; all writer types must be drained, not just public API requests.
- **RISK-003:** Computed counters add indexed database work; verify representative query plans without introducing caches preemptively.
- **RISK-004:** Anonymous star increments are not attributable to users; their removal is an intentional consequence of the selected signed-in-user semantics.
- **ASSUMPTION-001:** Preserve features and public response shapes except the explicitly selected star-request requirement.
- **ASSUMPTION-002:** Retain timestamps, identity keys, checksum semantics, existing policy deletion behavior, and authorization rules unless an implementation defect prevents the stated acceptance criteria.
- **ASSUMPTION-003:** Existing small table sizes make a transactional maintenance migration practical; rehearsal must validate the specified timeout budget.
- **ASSUMPTION-004:** Completion of local work does not imply production deployment or authorization to perform it.

## 8. Related Specifications / Further Reading

- Registry schema: `/Users/yonatan/Dev/Aptitude/registry/docs/reference/schema.md`
- Deployment procedure: `/Users/yonatan/Dev/Aptitude/registry/docs/architecture/render-neon-deployment.md`
- [PostgreSQL ALTER TABLE and locking behavior](https://www.postgresql.org/docs/17/sql-altertable.html)
- [Render maintenance mode and its traffic boundary](https://render.com/docs/maintenance-mode)
