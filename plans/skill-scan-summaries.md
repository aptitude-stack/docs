---
goal: Publish scan summaries and expand Maturity and Security metadata rows
version: "1.0"
date_created: "2026-09-06"
status: Planned
tags: [publisher, registry, website, migration]
---

# Introduction

![Status: Planned](https://img.shields.io/badge/status-Planned-blue)

Approved implementation plan: publish a compact sanitized assessment for each new skill version and expose it beneath independently expandable Maturity and Security rows. Execution status and verification are recorded separately in `skill-scan-summaries-verification.md` so this specification remains unchanged.

## 1. Requirements & Constraints

- **REQ-001**: Keep both rows independently collapsed initially with their labels and scores visible, full-row keyboard triggers, and trailing chevrons.
- **REQ-002**: Show Maturity validation/performance results and warnings, plus Security decisions, checks, and severity-labelled findings.
- **REQ-003**: Store nullable `metadata.assessment` on immutable `skill_versions` and return it through existing metadata APIs.
- **REQ-004**: Old versions keep their scores and show “Summary unavailable for this version”; no backfill.
- **SEC-001**: Allowlist fields, redact credentials and absolute paths, exclude raw evidence, transcripts, commands and environment dumps, and render public text inertly.
- **SEC-002**: Label summaries Publisher-reported; show review_required explicitly and never imply an empty finding list guarantees safety.
- **CON-001**: Preserve score formulas, gates, policy, confirmation, unrelated work, theme, typography and density; add no dependencies or catalog indicators.
- **CON-002**: No commits, package publication, hosted migrations or deployment are authorized by this plan.

### Public contract

`assessment` is nullable and optional. A present value contains `schema_version: 1`, UTC `assessed_at`, and these objects:

- `maturity`: `validation_passed`, `validation_score`, nullable `upskill_score`, `upskill_status`, `test_case_count`, `models_tested`, `models_omitted`, nullable `baseline_success_rate` and `skilled_success_rate`, deduplicated `warnings`, and `warnings_omitted`.
- `security`: `scanned`, nullable `decision` (`allow`, `review_required`, `block`), `checks_run`, `checks_omitted`, `findings` containing only `check`, `severity` (`low`, `medium`, `high`, `critical`), and `explanation`, plus `findings_omitted`.

Validate nested types, finite [0,1] scores/rates, nonnegative integer counts, and enums; forbid unknown fields. Limit text items to 1,000 characters and lists to 100 entries, reporting omitted counts. Generate the timestamp once, preserving it during receipt reuse. Explain the existing 50% validation / 50% Upskill formula, including zero contribution from unavailable performance evidence without portraying it as a measured zero.

## 2. Implementation Steps

### Implementation Phase 1 — Registry

- **GOAL-001**: Round-trip assessments through publish, storage and fetch; tasks run sequentially.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-001 | Extend Registry DTOs/core/record metadata with validated nullable assessment and verify legacy and invalid payload tests. | Planned | 2026-09-06 |
| TASK-002 | Add migration 0014 after 0013 and JSONB assessment on skill_versions and verify upgrade/downgrade preserves rows. | Planned | 2026-09-06 |
| TASK-003 | Update publish/fetch/repository mappings and verify a publish-to-fetch integration test. | Planned | 2026-09-06 |
| TASK-004 | Include non-null assessment in _version_checksum_digest and verify legacy digest preservation and changed-summary conflicts. | Planned | 2026-09-06 |

### Implementation Phase 2 — Publisher

- **GOAL-002**: Generate identical CLI/MCP summaries; depends on Phase 1 contract.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-005 | Extend DeliveryStage._build_registry_payload from existing evaluation evidence and verify exact assessment output. | Planned | 2026-09-06 |
| TASK-006 | Reuse credential redaction and remove absolute paths from allowlisted public messages and verify privacy fixtures. | Planned | 2026-09-06 |
| TASK-007 | Bump signed receipt schema and preserve assessment/timestamp during reuse and verify tampering/expiry/old-schema rejection. | Planned | 2026-09-06 |

### Implementation Phase 3 — Website

- **GOAL-003**: Expand metadata rows without additional requests; depends on Phase 1 contract.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-008 | Extend website metadata types and response validation and verify legacy responses and malformed assessments. | Planned | 2026-09-06 |
| TASK-009 | Add installed-Radix Collapsible composition to SkillMetadata and verify independent pointer/keyboard expansion. | Planned | 2026-09-06 |
| TASK-010 | Style inline results with theme tokens and verify mobile/desktop light/dark layout and focus/reduced-motion feedback. | Planned | 2026-09-06 |

### Implementation Phase 4 — Verification

- **GOAL-004**: Verify locally and document release order; depends on Phases 1–3.

| Task | Description | Status | Date |
|---|---|---|---|
| TASK-011 | Run Publisher/Registry pytest and website Bun tests/typecheck and resolve feature regressions. | Planned | 2026-09-06 |
| TASK-012 | Verify a fixture through Publisher payload, Registry persistence/fetch and website rows without external evaluators or hosted writes. | Planned | 2026-09-06 |
| TASK-013 | Document migration → Registry → Publisher/website rollout and legacy behavior with separate production approval. | Planned | 2026-09-06 |

## 3. Alternatives

- **ALT-001**: Raw reports expose unnecessary private information.
- **ALT-002**: Separate assessment tables/endpoints add unnecessary history/query machinery.
- **ALT-003**: Warnings-only UI was rejected in favor of results plus warnings.

## 4. Dependencies

- **DEP-001**: Existing SQLAlchemy/Alembic/Pydantic/PostgreSQL JSONB.
- **DEP-002**: Existing Publisher evidence, report redaction, signed receipts and artifact integrity checks.
- **DEP-003**: Installed Radix UI/Lucide, website tokens, Python/Bun test environments.

## 5. Files

- **FILE-001**: Registry metadata DTOs, core models/ports/checksum, repository mappings, skill_version model and new migration.
- **FILE-002**: Publisher delivery stage, public assessment helper, receipt version and related tests.
- **FILE-003**: Website SkillMetadata, ui/collapsible, types/client validation, globals.css and related tests.

## 6. Testing

- **TEST-001**: Round-trip clean, warning-bearing and review-required assessments.
- **TEST-002**: Legacy publication/fetch/checksums and null/absent summaries.
- **TEST-003**: Invalid types, bounds, unknown fields and non-finite numbers.
- **TEST-004**: Credential/path redaction, excluded raw evidence and inert HTML-like text.
- **TEST-005**: Signed summary/timestamp reuse, tampering, expiry and obsolete receipts.
- **TEST-006**: Independent expansion, keyboard/focus, long text, empty states, responsive themes.

## 7. Risks & Assumptions

- **RISK-001**: Publisher-reported evidence is not independently verified by Registry.
- **RISK-002**: Registry must support assessment before new Publisher payloads are sent.
- **ASSUMPTION-001**: New publications only, no backfill or rescans.
- **ASSUMPTION-002**: Current metadata lives on skill_versions after migration 0013, not the retired table.

## 8. Related Specifications / Further Reading

- [Metadata/cache plan](publisher-metadata-and-cache.md)
- [Database cleanup](db-structure-cleanup-and-alignment.md)
- [shadcn Radix Collapsible](https://ui.shadcn.com/docs/components/radix/collapsible)
