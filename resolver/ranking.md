# Aptitude Resolver - Ranking

Ranking is local to the resolver. The registry returns candidate slugs; the
resolver fetches metadata, filters illegal candidates, reranks the legal set,
and then selects one root skill.

## Ranking Flow

```text
POST /discovery
  -> fetch candidate metadata
  -> select preferred version per slug
  -> Phase 1 policy filter
  -> rerank remaining candidates
  -> final root selection
```

Ranking never sees candidates rejected by Phase 1 policy.

## Selection Profiles

The resolver supports three selection profiles:

| Profile | Bias |
| --- | --- |
| `balanced` | Relevance, trust/lifecycle, size, default/current signals, and stable tie-breakers. |
| `low-cost` | Prefer known and smaller token/content estimates while preserving relevance and legality. |
| `high-trust` | Prefer higher trust and lifecycle quality more strongly. |

Profiles are preferences, not policy. They cannot make an illegal candidate
legal.

## Candidate Signals

Current ranking uses signals from the parsed intent and selected version,
including:

- exact slug/name relevance,
- match reasons from registry discovery,
- tag/field match explanations,
- lifecycle status,
- trust tier,
- token estimate and content size when known,
- current-default version flag,
- semantic version order,
- publication time,
- slug/version stable tie-breakers.

The exact tuple differs by profile. The important invariant is deterministic
ordering for the same inputs, policy context, and profile.

## Version Selection

Before candidate reranking, each slug must resolve to one preferred version.
Version selection prefers:

1. current default,
2. lifecycle rank,
3. trust rank,
4. semantic version order,
5. published timestamp,
6. stable slug/version tie-breakers.

## Final Root Selection

After reranking, the resolver chooses exactly one root:

| Mode | Trigger |
| --- | --- |
| Explicit slug | Query directly matches a candidate slug. |
| Single candidate | Only one policy-compliant candidate remains. |
| Interactive choice | Multiple candidates remain and interaction is available/enabled. |
| Top-ranked fallback | Non-interactive mode, CI, MCP, or prompting disabled. |

Prompting is only allowed for root selection. Dependency resolution never
prompts.

## Dependency Resolution

Ranking applies only to the root candidate list. Dependencies are resolved from
authored selectors using deterministic version selection and graph validation.
The execution layer consumes the lockfile install order; it does not rerank.
