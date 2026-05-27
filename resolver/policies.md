# Aptitude Resolver — Policies

> How the resolver controls what is legal, what is preferred, and how decisions are made.

## Overview

The resolver separates governance into two distinct concerns that must never be conflated:

- **Policy** — hard rules about what is legal. Violations block an operation.
- **Selection preferences** — soft preferences about what is preferred among legal candidates. They shape ranking and interaction behavior but do not block anything.

Both are configured through a layered `aptitude.toml` system with environment variable overrides.

---

## Config Layers

Policy and selection preferences are assembled from multiple layers, applied in precedence order. Higher-priority layers always win; lower layers may only tighten constraints, never relax them.

### Policy Precedence (most → least authoritative)

1. Per-request override (CLI flags / MCP tool params)
2. Workspace `aptitude.toml` (nearest ancestor directory)
3. User `aptitude.toml`
4. System `aptitude.toml`
5. Resolver defaults

### Selection-Preference Precedence

1. CLI flag (`--prefer`, `--interaction-mode`)
2. Environment variable (`APTITUDE_PREFER`, `APTITUDE_INTERACTION_MODE`)
3. Workspace `aptitude.toml`
4. User `aptitude.toml`
5. System `aptitude.toml`
6. Resolver default

### Policy Merging Rules

Policy merging is **restrictive only**:

- `allowed_lifecycle_statuses` and `allowed_trust_tiers`: intersection of all active layers.
- `max_token_estimate`, `max_content_size_bytes`, `max_total_token_estimate`, `max_total_content_size_bytes`: minimum across all active layers.

A lower-priority layer may tighten but never weaken a higher-priority layer's policy. If a workspace sets `allowed_trust_tiers = ["verified"]` and the user config allows `["verified", "internal"]`, the effective value is `["verified"]`.

### Config File Locations

| Layer | Unix path | Windows path |
| --- | --- | --- |
| System | `/etc/aptitude/aptitude.toml` | `%PROGRAMDATA%\aptitude\aptitude.toml` |
| User | `~/.config/aptitude/aptitude.toml` | `%APPDATA%\aptitude\aptitude.toml` |
| Workspace | nearest `aptitude.toml` found by walking up from `cwd` | same |

`XDG_CONFIG_HOME` overrides the user config base directory on Unix.

### aptitude.toml Structure

```toml
[selection]
profile = "balanced"           # "balanced" | "low-cost" | "high-trust"
interaction_mode = "auto"      # "auto" | "always" | "never"

[policy]
allowed_lifecycle_statuses = ["published", "deprecated"]
allowed_trust_tiers = ["verified", "internal"]
max_token_estimate = 50000
max_content_size_bytes = 5242880      # 5 MiB
max_total_token_estimate = 200000
max_total_content_size_bytes = 20971520  # 20 MiB

[execution]
concurrent_downloads = 8
concurrent_installs = 4
```

All fields are optional. Omitted fields fall through to the next layer.

### Environment Overrides

| Variable | Effect |
| --- | --- |
| `APTITUDE_READ_TOKEN` | Registry bearer token |
| `APTITUDE_PREFER` | Selection profile override (`balanced`, `low-cost`, `high-trust`) |
| `APTITUDE_INTERACTION_MODE` | Interaction mode override |
| `APTITUDE_CONCURRENT_DOWNLOADS` | Execution download concurrency |
| `APTITUDE_CONCURRENT_INSTALLS` | Execution install concurrency |

---

## Policy Context

`PolicyContext` is the resolved hard-policy structure used for governance evaluation. It is built by merging all active layers and validated at construction time.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `profile` | `str` | `"default"` | Active policy profile name (informational) |
| `source` | `str` | `"client_default"` | Origin of the effective policy |
| `allowed_lifecycle_statuses` | `list[str]` | `["published", "deprecated", "archived"]` | Permitted version lifecycle states |
| `allowed_trust_tiers` | `list[str]` | `["verified", "internal", "untrusted"]` | Permitted trust tiers |
| `max_token_estimate` | `int \| None` | `None` | Per-skill token ceiling (`None` = no limit) |
| `max_content_size_bytes` | `int \| None` | `None` | Per-skill content size ceiling in bytes |
| `max_total_token_estimate` | `int \| None` | `None` | Aggregate graph token ceiling |
| `max_total_content_size_bytes` | `int \| None` | `None` | Aggregate graph content size ceiling |

Unknown values for `allowed_lifecycle_statuses` and `allowed_trust_tiers` are rejected at construction time with a `ValueError`.

---

## Policy Rules

Each governance check produces a `PolicyEvaluation` with `rule`, `passed`, `message`, and optionally the `SkillCoordinate` it applies to.

### Per-Skill Rules

| Rule | Check | Fail behavior |
| --- | --- | --- |
| `allowed_lifecycle_status` | `skill.lifecycle_status in policy.allowed_lifecycle_statuses` | Reject candidate / fail graph |
| `allowed_trust_tiers` | `skill.trust_tier in policy.allowed_trust_tiers` | Reject candidate / fail graph |
| `max_token_estimate` | `skill.token_estimate <= policy.max_token_estimate` | Reject. If `token_estimate` is `None` and a ceiling is configured, **fails closed** |
| `max_content_size_bytes` | `skill.content_size_bytes <= policy.max_content_size_bytes` | Reject. If `content_size_bytes` is `None` and a ceiling is configured, **fails closed** |

### Aggregate Rules (graph-level only)

| Rule | Check |
| --- | --- |
| `max_total_token_estimate` | Sum of all graph node token estimates ≤ ceiling. Fails if any node has `None` token estimate and a ceiling is configured. |
| `max_total_content_size_bytes` | Sum of all graph node content sizes ≤ ceiling. Fails if any node has `None` content size and a ceiling is configured. |

### Fallback / Unknown Value Semantics

| Field | No ceiling configured | Ceiling configured, value unknown |
| --- | --- | --- |
| `token_estimate = None` | ✓ passes | ✗ fails closed |
| `content_size_bytes = None` | ✓ passes | ✗ fails closed |
| `trust_tier = None` | Normalizes to `untrusted` | — |

---

## Two-Phase Governance

Governance runs in two sequential phases during fresh planning. Both phases must pass before a lockfile is written.

### Phase 1 — Candidate Policy Filtering

**When:** After the registry returns candidate slugs and before final root selection.

**What it does:** Evaluates every `DiscoveryCandidate` (at its selected version) against `PolicyContext`. Candidates that fail any per-skill rule are rejected with a `candidate_policy_reject` trace entry. Only policy-compliant candidates proceed to ranking and selection.

```
Registry candidate list
  → filter_policy_compliant_candidates()
  → [rejected candidates removed, trace entries emitted]
  → compliant candidate list → ranking → selection
```

**Key constraint:** Final ranking and root selection must compare only policy-compliant candidates. Ranking a blocked candidate before filtering is a layering violation.

### Phase 2 — Graph Governance

**When:** After the full dependency graph is resolved and before the lockfile is written.

**What it does:** Evaluates every node in the `ResolutionGraph` against `PolicyContext`, plus aggregate limits across the entire graph. Produces a list of `PolicyEvaluation` records.

```
ResolutionGraph
  → evaluate_resolution_graph()
  → per-node: lifecycle, trust, token, content-size
  → aggregate: total token estimate, total content size
  → [governance snapshot persisted in lockfile]
  → lock written only if all evaluations pass
```

A governance failure at Phase 2 does **not** silently fall through to another candidate. It must be surfaced explicitly as an error.

---

## Selection Preferences

`SelectionPreferences` captures soft user preferences that influence ranking and interaction behavior. Unlike `PolicyContext`, these cannot block an operation.

| Field | Values | Default | Description |
| --- | --- | --- | --- |
| `profile` | `"balanced"`, `"low-cost"`, `"high-trust"` | `"balanced"` | Ranking weight profile |
| `interaction_mode` | `"auto"`, `"always"`, `"never"` | `"auto"` | Whether to interactively prompt on ambiguity |
| `profile_source` | string | `"default"` | Where the profile value came from |
| `interaction_mode_source` | string | `"default"` | Where the interaction mode came from |

### Selection Profiles

| Profile | Ranking emphasis |
| --- | --- |
| `balanced` | Equal weight across trust, lifecycle freshness, and size |
| `low-cost` | Prefers lower token estimate and smaller content size |
| `high-trust` | Strongly prefers `verified` → `internal` → `untrusted` ordering |

### Interaction Modes

| Mode | Behavior |
| --- | --- |
| `auto` | Prompts when ambiguity remains and the session supports interactive input (CLI wizard). Silent selection otherwise. |
| `always` | Must prompt. Fails clearly if prompting is required but unavailable (e.g. MCP, CI). |
| `never` | Always chooses the top-ranked legal candidate deterministically without prompting. |

**Interaction scope:** Prompting is allowed only for root candidate ambiguity. Recursive dependency resolution must never prompt, regardless of interaction mode.

---

## Trust Tiers and Lifecycle (Ranking Helpers)

The resolver uses deterministic numeric ranks for tie-breaking during version selection.

**Trust tiers:**

| Tier | Rank |
| --- | --- |
| `verified` | 2 (highest) |
| `internal` | 1 |
| `untrusted` | 0 |
| unknown / `None` | -1 |

**Lifecycle statuses:**

| Status | Rank |
| --- | --- |
| `published` | 2 (highest) |
| `deprecated` | 1 |
| `archived` | 0 |
| unknown / `None` | -1 |

These ranks are used purely for ordering among already-legal candidates. They do not override policy decisions.

---

## Checksum Policy

The resolver enforces artifact integrity at materialization time.

- Algorithm: `sha256` (Phase 1).
- The registry publishes checksum facts in skill metadata; the resolver reads them from the lockfile.
- Verification happens on **compressed artifact bytes** before archive extraction.
- A mismatch raises `ContentChecksumMismatchError` and aborts the install. No partial installs are permitted.

---

## Policy Snapshot in Lockfile

The active policy context and selection preferences are persisted with every lockfile as `PolicySnapshot` and `SelectionSnapshot`. This allows auditing what rules were active at planning time, independent of current config state.

The governance evaluation results from both phases are persisted as `GovernanceSnapshotEntry` records (rule, passed, message, optional node_id). These are informational — they are not re-evaluated during lock replay.

---

## Inspecting Effective Policy

```bash
aptitude policy show
```

This command loads and merges all config layers and prints the effective `PolicyContext`, `SelectionPreferences`, and config-layer provenance (which layer each value came from).

Via MCP:

```json
{ "tool": "aptitude_show_policy" }
```
