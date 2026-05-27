# Aptitude Resolver — Ranking

> How the resolver ranks candidates, selects versions, and chooses a final root skill.

## Overview

Ranking in the resolver is a three-stage pipeline that runs during fresh planning, after the registry returns an ordered candidate list and after policy filtering removes ineligible versions:

```
POST /discovery → ordered slug list
  → candidate version resolution (fetch metadata per slug)
  → Phase 1 policy filtering (remove lifecycle / trust / size violations)
  → candidate reranking (profile-aware, client-side)
  → version selection (per-slug deterministic ordering)
  → final root selection (explicit / single / interactive / top-ranked)
```

The registry's ordering is advisory — it reflects server-side signals (lexical match, semantic similarity, co-usage lift). The resolver then applies its own client-side ranking on top, shaped by the active selection profile. Final ranking is always deterministic for the same inputs and profile.

---

## Stage 1 — Candidate Reranking

`discovery/reranking/candidate_reranker.py` reranks the policy-compliant candidate list before final root selection. The reranker does **not** make the final selection decision; it shapes the ordered list from which the selection step picks.

### RankingComponents

Each candidate is reduced to a `RankingComponents` dataclass before sorting. These fields are computed from the candidate's `SearchIntent` (the parsed query) and the selected `VersionSummary` (the best version for that slug):

| Field | Type | Description |
| --- | --- | --- |
| `is_exact_slug_match` | `bool` | Slug is an exact string match against the query name |
| `is_exact_name_match` | `bool` | Normalized skill name equals the normalized query name |
| `trust_tier_rank` | `int` | Numeric rank of the selected version's trust tier (see table below) |
| `lifecycle_rank` | `int` | Numeric rank of the selected version's lifecycle status |
| `has_description` | `bool` | Skill has a non-empty description |
| `tag_overlap_count` | `int` | Number of query tags present in the skill's tag set |
| `token_estimate` | `int \| None` | Token estimate of the selected version (`None` = unknown) |
| `content_size_bytes` | `int \| None` | Content size in bytes of the selected version |
| `usage_count` | `int` | Cumulative install/usage count for the skill |
| `is_current_default` | `bool` | The selected version is the slug's current default version |
| `published_at` | `datetime \| None` | Publication timestamp of the selected version |
| `query_term_matches` | `int` | Number of query terms found in slug + name + description |
| `slug` | `str` | Skill slug (used as a stable tie-breaker) |
| `version` | `str` | Selected version string (used as a stable tie-breaker) |

### Profile Sort Keys

The active selection profile determines field priority. All sort keys are descending unless noted as ascending (↑).

**`balanced`** — equal weight across trust, freshness, and size:

1. `is_exact_slug_match` (exact slug match first)
2. `is_exact_name_match`
3. `trust_tier_rank`
4. `lifecycle_rank`
5. `tag_overlap_count`
6. `query_term_matches`
7. `usage_count`
8. `has_description`
9. `is_current_default`
10. `published_at`
11. `token_estimate` ↑ (prefer smaller — `None` sorts last)
12. `content_size_bytes` ↑ (prefer smaller — `None` sorts last)
13. `slug` ↑ (ascending, stable tie-breaker)
14. `version` ↑ (ascending, stable tie-breaker)

**`low-cost`** — prefer lower token estimate and smaller content:

1. `is_exact_slug_match`
2. `is_exact_name_match`
3. `token_estimate` ↑ (prefer smallest — `None` sorts last)
4. `content_size_bytes` ↑ (prefer smallest — `None` sorts last)
5. `trust_tier_rank`
6. `lifecycle_rank`
7. `tag_overlap_count`
8. `query_term_matches`
9. `usage_count`
10. `has_description`
11. `is_current_default`
12. `published_at`
13. `slug` ↑
14. `version` ↑

**`high-trust`** — strongly prefer verified → internal → untrusted:

1. `is_exact_slug_match`
2. `is_exact_name_match`
3. `trust_tier_rank` (strongest weight — `verified` always beats `internal`)
4. `lifecycle_rank`
5. `tag_overlap_count`
6. `query_term_matches`
7. `usage_count`
8. `is_current_default`
9. `published_at`
10. `has_description`
11. `token_estimate` ↑
12. `content_size_bytes` ↑
13. `slug` ↑
14. `version` ↑

The `high-trust` profile differs from `balanced` in that trust tier is placed above lifecycle and all content-quality signals, while token and content size are demoted to near-last.

---

## Stage 2 — Version Selection

Before reranking, each candidate slug must have exactly one version selected. `resolution/solver/version_selection.py` applies a deterministic multi-field sort over all available versions for a given slug.

### `select_preferred_version` Sort Key

Versions are sorted descending on the following tuple (earlier fields take priority):

1. `is_current_default` — the slug's marked default version is always preferred
2. `lifecycle_rank` — `published` > `deprecated` > `archived`
3. `trust_rank` — `verified` > `internal` > `untrusted`
4. `Version(version)` — semantic version ordering (higher version preferred)
5. `published_at` — more recent publication preferred (`None` sorts last)
6. `slug` (ascending) — stable tie-breaker across slugs
7. `version` (ascending) — stable string tie-breaker for identical semantic versions

This ordering ensures that the current default, most stable, highest-trust, and most recent version wins deterministically. Version selection runs before reranking — `RankingComponents` reflects the already-selected best version.

---

## Stage 3 — Final Root Selection

`resolution/solver/candidate_selection.py` picks exactly one candidate from the reranked list. The selection mode is determined by query shape and interaction settings:

| Mode | Trigger | Behavior |
| --- | --- | --- |
| `explicit_slug` | Query matches an exact slug in the candidate list | Immediately selects that candidate; no ranking comparison needed |
| `single_candidate` | Only one policy-compliant candidate remains | Selects it without prompting |
| `interactive_choice` | Multiple candidates remain and `interaction_mode` is `auto` (with interactive session) or `always` | Presents a ranked list to the user via the CLI wizard |
| `non_interactive_top_ranked` | Multiple candidates remain and prompting is unavailable or disabled (`never` / MCP / CI) | Selects the top-ranked candidate deterministically |

**Interaction constraint:** Prompting is allowed only for root candidate selection. Recursive dependency resolution must never prompt, regardless of the configured interaction mode.

---

## Trust Tier and Lifecycle Ranks

These numeric values are used throughout both version selection and candidate reranking as deterministic tie-breakers. They reflect preference, not policy — a lower rank does not block a candidate (that is handled by Phase 1 policy filtering).

**Trust tiers:**

| Tier | Rank |
| --- | --- |
| `verified` | 2 |
| `internal` | 1 |
| `untrusted` | 0 |
| unknown / `None` | -1 |

**Lifecycle statuses:**

| Status | Rank |
| --- | --- |
| `published` | 2 |
| `deprecated` | 1 |
| `archived` | 0 |
| unknown / `None` | -1 |

---

## Explainability

The resolver attaches human-readable explanations to every ranked candidate. These fields appear on `DiscoveryCandidate` and are preserved through to the lockfile and MCP/CLI output.

### `selection_reason`

A short human-readable string describing why this candidate was chosen or ranked where it was. Generated by `_selection_reason()` in the reranker. Examples:

- `"Exact slug match"` — query name matched the slug directly
- `"Exact name match"` — query name matched the normalized skill name
- `"Top-ranked by trust tier"` — `high-trust` profile, `verified` tier won
- `"Top-ranked by cost"` — `low-cost` profile, smallest token estimate

### `_comparison_reason()`

For interactive mode, the reranker produces a per-pair comparison note explaining why one candidate was placed above another in the ranked list. This is surfaced in the CLI wizard so the user understands the ordering before confirming a selection.

### `match_reasons`

Set of string tags indicating which signals contributed to the match: `exact_slug_match`, `exact_name_match`, `text_match`, `tag_filter_match`, `structured_filter_match`. These come from the registry's explanation fields and are passed through the resolver unchanged.

### `matched_labels`

The intersection of the query's requested tags and the skill's actual tags, carried through to the `DiscoveryCandidate` for display.

---

## Ranking in Context

Ranking operates strictly within the policy boundary established by Phase 1 governance filtering. The reranker only ever sees policy-compliant candidates — it has no visibility into rejected candidates and must not attempt to recover them.

After final root selection, ranking plays no further role. Recursive dependency resolution uses `select_preferred_version` for version choice but does not rerank or prompt. Execution consumes `lockfile.install_order` directly and does not re-evaluate ranking signals.

See [`policies.md`](policies.md) for the two-phase governance model and [`architecture.md`](architecture.md) for where ranking fits in the full fresh-planning data flow.
