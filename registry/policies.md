# Aptitude Registry — Governance Policies

> How the registry controls who can publish, read, and manage skill versions.

## Overview

The registry enforces a multi-dimensional governance model on every operation. A request is only allowed when all of the following align:

1. The caller holds the required **scope** on their token.
2. The caller holds the required **namespace grant** for the target namespace and promotion channel.
3. The requested operation is consistent with the **trust-tier publish rules** (for publish) or the **lifecycle transition rules** (for status updates).
4. The version's current **enterprise workflow state** — review state, promotion channel, policy pack — permits visibility or mutation.

Governance is evaluated in the `GovernancePolicy` class (`app/core/governance.py`) against a `PolicyProfile` resolved from settings.

---

## Caller Scopes

Every service token carries a set of scopes. Scopes grant access to route categories.

| Scope | Grants |
| --- | --- |
| `read` | Discovery, resolution, exact metadata and content fetch, version listing, catalog |
| `publish` | Skill publication (subject to trust-tier rules) |
| `review` | Governance updates: review state, promotion channel, trust-tier re-classification, trust-evidence append |
| `admin` | All of the above, plus lifecycle transitions, enterprise admin endpoints, and global namespace access |
| `telemetry` | Star event submission and user-star listing |

`admin` is a superset: a token with `admin` implicitly satisfies any `read`, `publish`, or `review` check.

---

## Trust Tiers

Every version is assigned a trust tier at publish time. The tier is immutable from the artifact's perspective — changing it after publish requires a `PATCH .../governance` operation, which is an enterprise workflow update recorded in the audit log, not a re-write of artifact bytes.

| Tier | Meaning | Required scope to publish | Provenance required |
| --- | --- | --- | --- |
| `untrusted` | Community or unevaluated skill | `publish` | No |
| `internal` | First-party or organization-verified skill | `publish` | **Yes** |
| `verified` | Registry-certified skill with full provenance chain | `admin` | **Yes** |

A caller with only `publish` scope cannot publish at the `verified` tier. Attempting to do so raises `POLICY_PUBLISH_FORBIDDEN`.

---

## Lifecycle States

Every version has a `lifecycle_status`. Transitions are admin-gated and policy-enforced.

```
published ──► deprecated ──► archived
    │                           ▲
    └───────────────────────────┘
           (direct archive)

deprecated ──► published  (re-activation)
```

| From | Allowed targets |
| --- | --- |
| `published` | `deprecated`, `archived` |
| `deprecated` | `published`, `archived` |
| `archived` | *(none — terminal state)* |

Attempting a disallowed transition raises `POLICY_STATUS_TRANSITION_FORBIDDEN`.

**Visibility by lifecycle status:**

| Status | Default discovery | Read-scope exact fetch | Admin exact fetch |
| --- | --- | --- | --- |
| `published` | ✓ | ✓ | ✓ |
| `deprecated` | ✗ (must be explicitly requested) | ✓ | ✓ |
| `archived` | ✗ | ✗ | ✓ |

Read-scope callers may request `deprecated` versions explicitly in discovery, but never `archived`. Admin callers may read all states.

---

## Namespaces and Namespace Grants

Skills belong to exactly one namespace. Namespaces are owned by organizations and have a `visibility` (`public` or `private`).

A `CallerIdentity` carries zero or more `NamespaceGrant` objects. Each grant specifies:

- `namespace`: the target namespace slug, or `*` for global access.
- `roles`: a set of `NamespaceRole` values (`read`, `publish`, `review`, `admin`).
- `promotion_channels`: a set of allowed channels (`dev`, `staging`, `prod`, or `*`).

**Fallback rules for tokens without namespace grants:**

- A token with `admin` scope but no explicit grants is treated as having global access to all namespaces and channels.
- A token with only `publish` or `read` scope and no grants is confined to the `public` namespace on the `prod` channel.

Operations that span namespaces (e.g. moving a skill with `PATCH /admin/skills/{slug}/ownership`) require a grant that covers both the source and target namespaces, or an admin-scope token with global access.

---

## Promotion Channels

Promotion channels are a governance concept, not the same as `APP_ENV` (the deployment environment).

| Channel | Typical use |
| --- | --- |
| `dev` | Early development or untrusted import, not shown in production catalogs |
| `staging` | Pre-release review, visible to reviewers but not standard readers |
| `prod` | Production-ready, visible to all callers with sufficient namespace access |

**Default publish behavior:**

- `internal` artifacts default to `prod` channel and `approved` review state.
- `imported` artifacts (from an external source) default to `dev` channel and `pending_review` review state. They are hidden from production readers until reviewed and promoted.

---

## Review States

| State | Meaning |
| --- | --- |
| `pending_review` | Imported or flagged; not visible to standard `read` callers |
| `approved` | Default for internally published skills; visible under normal governance |
| `rejected` | Blocked from production; not visible to standard `read` callers |

Only callers holding a `review` namespace grant can see `pending_review` and `rejected` versions in listings or exact reads.

---

## Artifact Origins

| Origin | Meaning |
| --- | --- |
| `internal` | Published directly by a first-party or organization publisher |
| `imported` | Ingested from an external source; starts in `pending_review` / `dev` |
| `verified` | Passed a full verification pipeline |
| `restricted` | Subject to policy-pack access controls |

Origin is advisory metadata; it determines default review state and promotion channel and can be updated via the governance endpoint.

---

## Policy Packs

A policy pack is a named JSON rule document attached to a version. Packs extend the default visibility model with additional access controls.

A policy pack is created or updated via `PUT /admin/policy-packs/{slug}` and attached to a version at publish time or via `PATCH .../governance`.

**Supported rules keys:**

| Key | Type | Effect |
| --- | --- | --- |
| `visibility` | `"public"` \| `"restricted"` | When `restricted`, only explicitly allowed callers can see this version |
| `requires_verified_publisher` | `bool` | When `true`, only `verified` trust-tier versions are visible |
| `allowed_token_ids` | `string[]` | Explicit token ID allowlist for `restricted` visibility |
| `allowed_namespaces` | `string[]` | Namespace allowlist for `restricted` visibility |

A `POLICY_PACK_FORBIDDEN` violation is raised when a caller tries to read a version whose policy pack blocks them. This check applies consistently across discovery, catalog search, version listing, exact metadata, and exact content.

---

## Provenance Metadata

Provenance is required for `internal` and `verified` trust tiers. It is validated and normalized at publish time.

| Field | Validation |
| --- | --- |
| `repo_url` | Non-empty text |
| `commit_sha` | Hex string, 7–64 characters |
| `tree_path` | Optional non-empty text |
| `publisher_identity` | Optional non-empty text |

The active policy profile name is automatically appended as `policy_profile` in the stored provenance. Provenance cannot be modified after publish; it is part of the immutable version record.

---

## Error Codes

| Code | HTTP status | Description |
| --- | --- | --- |
| `AUTHENTICATION_REQUIRED` | 401 | No bearer token provided |
| `MALFORMED_AUTH_TOKEN` | 401 | Token format invalid |
| `INVALID_AUTH_TOKEN` | 401 | Token not found |
| `INACTIVE_AUTH_TOKEN` | 401 | Token has been revoked |
| `EXPIRED_AUTH_TOKEN` | 401 | Token past expiry |
| `INSUFFICIENT_SCOPE` | 403 | Token lacks the required scope for this route |
| `POLICY_PUBLISH_FORBIDDEN` | 403 | Trust-tier publish rule not satisfied |
| `POLICY_PROVENANCE_REQUIRED` | 403 | Provenance missing for trust tier that requires it |
| `POLICY_PROVENANCE_INVALID` | 422 | Provenance field values failed validation |
| `POLICY_NAMESPACE_FORBIDDEN` | 403 | Caller lacks the required namespace grant |
| `POLICY_NAMESPACE_INVALID` | 422 | Namespace slug failed format validation |
| `POLICY_STATUS_TRANSITION_FORBIDDEN` | 403 | Lifecycle transition is not permitted |
| `POLICY_EXACT_READ_FORBIDDEN` | 403 | Version lifecycle state not readable by caller |
| `POLICY_REVIEW_STATE_FORBIDDEN` | 403 | Caller lacks review scope for this review state |
| `POLICY_DISCOVERY_FORBIDDEN` | 403 | Requested lifecycle status not allowed in discovery |
| `POLICY_PACK_FORBIDDEN` | 403 | Policy pack denies access to this version |
| `POLICY_PROFILE_INVALID` | 500 | Policy profile misconfiguration (boot-time check) |

---

## Default Policy Profile

The built-in `default` profile applies when no other profile is configured.

```python
PolicyProfile(
    name="default",
    publish_rules={
        "untrusted": PublishRule(required_scope="publish", provenance_required=False),
        "internal":  PublishRule(required_scope="publish", provenance_required=True),
        "verified":  PublishRule(required_scope="admin",   provenance_required=True),
    },
    lifecycle_transitions={
        "published":  ("deprecated", "archived"),
        "deprecated": ("published", "archived"),
        "archived":   (),
    },
    discovery_default_statuses=("published",),
    discovery_read_statuses=("published", "deprecated"),
    discovery_admin_statuses=("published", "deprecated", "archived"),
    exact_read_statuses=("published", "deprecated"),
)
```

The profile is injected into `GovernancePolicy` at service startup. All governance evaluations reference the profile rather than hardcoded values, making it possible to swap profiles via configuration for multi-tenant or enterprise deployments.
