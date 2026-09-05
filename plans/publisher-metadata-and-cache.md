---
goal: Separate Publisher source metadata from cached evaluation output
version: "1.0"
date_created: "2026-09-05"
last_updated: "2026-09-05"
owner: Aptitude
status: Completed
tags: [publisher, refactoring, metadata, cache]
---

# Introduction

![Status: Completed](https://img.shields.io/badge/status-Completed-brightgreen)

Move Aptitude publishing metadata into `aptitude.yaml` and replace `.publisher_artifacts/` with one cached JSON report per skill.

## 1. Requirements & Constraints

- **REQ-001**: Require `aptitude.yaml` beside `SKILL.md`; do not fall back to Aptitude metadata in Markdown frontmatter.
- **REQ-002**: Keep `name`, `description`, optional standard frontmatter fields, and instructions in `SKILL.md`; leave `agents/openai.yaml` untouched.
- **REQ-003**: Retain one latest JSON report per canonical skill directory, containing stage results, gate decisions, normalized evaluation evidence, and the signed inspection receipt when available.
- **REQ-004**: Create all evaluator working files in temporary storage outside the source directory and clean them after success or failure.
- **REQ-005**: Preserve Registry request shapes, evaluation decisions, explicit publication confirmation, and receipt freshness/integrity checks.
- **SEC-001**: Reuse receipt redaction and signing; never persist credentials, raw environment dumps, or unredacted evaluator transcripts.
- **CON-001**: Preserve existing `.publisher_artifacts` directories but stop reading or writing them; continue excluding them from inventory and bundles.
- **CON-002**: Do not change Registry, Resolver, installed user skills, historical evaluation outputs, or published versions.
- **CON-003**: Use existing PyYAML and Python standard-library facilities; add no dependencies.

### Source metadata contract

Use a flat `aptitude.yaml` mapping:

```yaml
version: "0.1.0"
intent: create_skill
tags:
  - python
inputs_schema: {}
outputs_schema: {}
relationships:
  depends_on:
    - slug: python-testing
      version: "0.1.2"
```

- Preserve current identity override precedence: explicit CLI/MCP values override the file; otherwise read version and intent from `aptitude.yaml`, and slug from `SKILL.md` name.
- Preserve current required-field and value validation for tags and schemas.
- Relationships are optional; omitted families default to empty lists.
- Preserve support for currently recognized authored `token_estimate`, `maturity_score`, and `security_score` fields without changing how computed scores supersede them.
- Reject duplicate YAML keys, unknown sidecar fields, invalid types, and malformed relationship selectors with file-specific errors.
- Do not duplicate `name`, `description`, `license`, or `compatibility` in the sidecar.
- Reject known Aptitude fields remaining in legacy frontmatter with migration guidance; unrelated standard `metadata` entries remain valid.
- Include `aptitude.yaml` in the skill bundle and its checksum so metadata edits invalidate inspection reuse.

### Cache contract

- Cache root: absolute `XDG_CACHE_HOME` when configured; otherwise `~/.cache`; append `aptitude/publisher`.
- Report path: `<cache-root>/<sha256(canonical-absolute-skill-directory)>.json`.
- JSON envelope: `schema_version: 1`, `skill_root`, `updated_at`, `status`, `stages`, `gates`, `evidence`, `warnings`, `error`, and `inspection_receipt`.
- Report status: `running`, `ready`, `blocked`, or `failed`; retain exact evaluation decisions inside evidence.
- Keep the existing receipt schema, MAC, expiry, configuration fingerprint, identity, governance, and source-digest validation inside `inspection_receipt`.
- Atomically replace the report using a same-directory temporary file; use owner-only file permissions.
- On a new evaluation, clear the previous receipt; checkpoint after each completed stage/gate and finalize on failure.
- Corrupt or expired cached evidence triggers reevaluation. Cache write failures produce an actionable error; never fall back to writing under the skill.
- Concurrent runs use isolated temporary directories; the last atomic report write wins. Publishing uses its own validated in-memory receipt.
- Store normalized diagnostics, not raw evaluator output. Clear temporary-file references before persistence.
- No history, automatic migration of old artifacts, or cache-management CLI in this change.

## 2. Implementation Steps

Paths below are relative to `/Users/yonatan/Dev/Aptitude`.

### Implementation Phase 1

- **GOAL-001**: Load and validate publishing metadata exclusively from `aptitude.yaml`.
- Tasks are sequential.

| Task | Description | Status | Date |
|------|-------------|--------|------|
| TASK-001 | Add `publisher/publisher/manifest.py` with `load_manifest()` using safe YAML parsing and explicit field validation, verified by valid, missing, duplicate-key, and malformed-input tests. | Complete | 2026-09-05 |
| TASK-002 | Update `DiscoveryStage.run`, `IdentityStage._populate_identity_from_skill`, and `MetadataStage._load_metadata_payload` under `publisher/publisher/stages/` to share the parsed manifest and preserve override precedence, verified by source-selection tests. | Complete | 2026-09-05 |
| TASK-003 | Update `publisher/publisher/relationships.py`, `ValidationStage`, and `DeliveryStage._build_registry_payload` to use manifest relationships and reject legacy Aptitude frontmatter, verified by unchanged Registry payload assertions. | Complete | 2026-09-05 |
| TASK-004 | Verify `publisher/publisher/artifacts/bundle.py:build_bundle_bytes` includes the sidecar and preserves `agents/openai.yaml`, with tests proving either file changes the source digest. | Complete | 2026-09-05 |

### Implementation Phase 2

- **GOAL-002**: Persist one report outside the skill directory across every Publisher entry point.
- Depends on Phase 1; tasks are sequential.

| Task | Description | Status | Date |
|------|-------------|--------|------|
| TASK-005 | Add cache-path and atomic-report helpers in `publisher/publisher/artifacts/report.py` and update `PublishContext` in `publisher/publisher/domain/models.py`, verified by path isolation and atomic-write tests. | Complete | 2026-09-05 |
| TASK-006 | Update `PublisherPipeline.run` in `publisher/publisher/app/pipeline.py` and stage implementations to checkpoint the shared report instead of writing numbered artifacts, verified by successful and blocked-run filesystem assertions. | Complete | 2026-09-05 |
| TASK-007 | Update adapters in `publisher/publisher/integrations/` to run evaluators against a temporary skill copy with temporary working directories and absolute configured input paths, verified by timeout and failure cleanup tests. | Complete | 2026-09-05 |
| TASK-008 | Remove the redundant persisted delivery-JSON compression stage from `publisher/publisher/stages/compression.py` and its pipeline/model consumers while retaining actual bundle compression in `artifacts/bundle.py`, verified by upload-bundle tests. | Complete | 2026-09-05 |
| TASK-009 | Update `publisher/publisher/interfaces/mcp/receipt.py` and `server.py` to store and retrieve the signed receipt inside the report, verified by existing tamper, expiry, configuration-change, and source-change tests. | Complete | 2026-09-05 |
| TASK-010 | Update CLI, menu, and MCP response models under `publisher/publisher/app/` and `publisher/publisher/interfaces/mcp/` to expose `report_path` instead of `artifacts_dir`, verified by entry-point response tests. | Complete | 2026-09-05 |

### Implementation Phase 3

- **GOAL-003**: Provide migration guidance and prove source directories remain unchanged.
- Depends on Phases 1–2.

| Task | Description | Status | Date |
|------|-------------|--------|------|
| TASK-011 | Migrate maintained Publisher test fixtures and current examples under `publisher/` to `aptitude.yaml` while preserving historical `runs/` content, verified by fixture discovery tests. | Complete | 2026-09-05 |
| TASK-012 | Update `publisher/README.md`, `publisher/PYPI.md`, and current documents under `docs/publisher/` with the sidecar contract, cache location, response-field change, and manual migration example, verified by searching for obsolete instructions. | Complete | 2026-09-05 |
| TASK-013 | Run `UV_CACHE_DIR=.uv-cache uv run --extra dev python -m pytest` from `publisher/` and inspect diffs in affected repositories, marking completion only after local checks pass. | Complete | 2026-09-05 |

## 3. Alternatives

- **ALT-001**: JSON is stricter but lacks comments; YAML reuses an installed dependency and suits nested relationships.
- **ALT-002**: TOML has a standard-library parser but is less convenient for nested schemas.
- **ALT-003**: Extending `agents/openai.yaml` would mix Aptitude publishing fields with OpenAI-specific configuration without documented schema support.
- **ALT-004**: Legacy frontmatter fallback was rejected in favor of one authoritative metadata source.
- **ALT-005**: Per-run reports and raw-output archives were rejected in favor of one latest report per skill.

## 4. Dependencies

- **DEP-001**: Existing PyYAML, dataclasses, JSON, hashlib, pathlib, and tempfile.
- **DEP-002**: Existing inspection-receipt signing, redaction, and freshness checks.
- **DEP-003**: Existing evaluator adapters and deterministic bundle construction.

## 5. Files

- **FILE-001**: Publisher manifest loading, domain models, stages, pipeline, artifact helpers, and evaluator adapters identified in Phases 1–2.
- **FILE-002**: CLI/menu and MCP response, receipt, and orchestration code identified in Phase 2.
- **FILE-003**: Publisher unit tests, maintained fixtures/examples, and current documentation identified in Phase 3.

## 6. Testing

- **TEST-001**: Accept a minimal valid sidecar; reject missing, malformed, duplicate-key, unknown-field, and incorrectly typed sidecars before external evaluation.
- **TEST-002**: Preserve override precedence, relationship normalization, omitted-list defaults, and Registry payload values.
- **TEST-003**: Reject legacy Aptitude frontmatter while accepting standard skill frontmatter and independent OpenAI configuration.
- **TEST-004**: Assert byte-for-byte unchanged source trees after inspect and mocked publish, including read-only source directories.
- **TEST-005**: Assert exactly one retained report per skill after repeated runs and no evaluator working files after handled success, failure, or timeout.
- **TEST-006**: Verify separate cache keys for equal-named skills in different directories and equal keys for symlink aliases of one directory.
- **TEST-007**: Verify receipt reuse, tampering rejection, expiry, credential/configuration changes, sidecar edits, and concurrent-write integrity.
- **TEST-008**: Verify failure reports retain normalized diagnostics without secrets or dead temporary paths.
- **TEST-009**: Exercise CLI, menu, and MCP with mocked evaluators and upload clients; no paid evaluations or hosted writes are required.

## 7. Risks & Assumptions

- **RISK-001**: Requiring the sidecar and replacing MCP `artifacts_dir` with `report_path` are intentional breaking changes; document both.
- **RISK-002**: Removing raw transcripts reduces forensic detail; normalized evidence and failure messages remain available.
- **RISK-003**: Forced process termination can leave temporary files; normal exception and timeout paths must clean them.
- **ASSUMPTION-001**: One report means one retained report per skill directory, not one global file.
- **ASSUMPTION-002**: Sidecar metadata remains in downloaded bundles for portability; moving it does not guarantee that every agent will avoid reading it.
- **ASSUMPTION-003**: Existing cache receipts are disposable; the first run after upgrading may repeat evaluation.
- **ASSUMPTION-004**: No commits, package publication, production writes, or changes to user-authored skills outside maintained Publisher fixtures/examples are included.

## 8. Related Specifications / Further Reading

- [Publisher skill format](/Users/yonatan/Dev/Aptitude/docs/publisher/skill-format.md)
- [Publisher audit pipeline](/Users/yonatan/Dev/Aptitude/docs/publisher/audit-pipeline.md)
- [OpenAI skill format and optional metadata](https://learn.chatgpt.com/docs/build-skills#optional-metadata): `SKILL.md` retains name and description; `agents/openai.yaml` configures appearance, invocation policy, and tool dependencies.

Implementation verification (2026-09-05): 178 tests passed; Publisher and documentation diffs pass `git diff --check`. Review findings for invalid UTF-8 and menu sidecar fallback were fixed and regression-tested. No commits or hosted writes were performed.
