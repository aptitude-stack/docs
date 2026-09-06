# Scan summaries execution record

| Tasks | Status | Date |
|---|---|---|
| TASK-001–004 Registry | Complete | 2026-09-06 |
| TASK-005–007 Publisher | Complete | 2026-09-06 |
| TASK-008–010 Website | Complete | 2026-09-06 |
| TASK-011–013 Verification and rollout | Complete | 2026-09-06 |

## Rollout

1. After local verification and separate production approval, verify the selected Neon project/branch and current Alembic revision before any hosted migration.
2. Apply migration 0014, then deploy Registry support for nullable assessments.
3. Release Publisher and website support only after Registry accepts the new metadata field.
4. Verify a newly approved publication returns its assessment and renders both expanded rows; existing versions show the unavailable-summary fallback.

Do not backfill old versions, change scores or gates, or create extra assessment tables. Reverting the UI/Publisher is safe while leaving the additive column in place; do not downgrade a populated assessment column merely to roll back an application release.

## Local verification

Publisher full suite: **242 passed**, using `UV_CACHE_DIR=.uv-cache uv run --no-sync --extra dev python -m pytest -q` from `publisher/`. Workspace contract checks confirm Registry accepts Publisher output without dropping assessment and Resolver accepts the extended response. Signed-receipt reuse preserves the exact summary and timestamp without repeating evaluation.

Registry full suite: **325 passed, 91 skipped**, using `env -u OPENAI_API_KEY UV_CACHE_DIR=.uv-cache uv run --no-sync --extra dev python -m pytest -q` from `registry/`. The skips are PostgreSQL-gated tests after the isolated test database was stopped. Separately, isolated local PostgreSQL verification passed **3 migration tests and 16 exact-fetch integration tests**, including assessment persistence and retrieval. The 22 focused assessment contract tests cover strict types, bounds, timestamp/schema version, unknown fields, and legacy/changed checksums. Ruff and mypy passed. Unsetting OPENAI_API_KEY avoids three existing tests inheriting an unrelated configured key; no credential files were inspected.

Website: **24 suites / 159 tests**, typecheck, and webpack production build passed (`bun run test --runInBand`, `bun run typecheck`, `bun run build --webpack` from `website/`). Validation includes blank strings and Unicode code-point boundaries matching Python. Turbopack's sandbox port-binding restriction was avoided using the supported webpack builder.

End-to-end fixture: generated the request and bundle with Publisher stages, posted through a temporary local Registry API backed by PostgreSQL, verified stored JSONB directly, fetched version metadata, and replayed that public response to the website. Browser checks confirmed default collapse, independent expansion, Enter/Space controls, light/dark rendering, and mobile wrapping. A discovered grid minimum-width issue was fixed at the detail-page/grid root; measured panel width is 348px within the 390px viewport. Temporary servers were stopped and browser theme/viewport restored.

Final read-only review reported no unresolved concrete correctness/security findings. All three repository diffs pass whitespace checks. No commits, hosted writes, production migration, package release, or deployment occurred.

The normal Publisher uv invocation attempted a blocked PyPI build dependency download; tests used the preinstalled environment via `uv run --no-sync --extra dev`.
