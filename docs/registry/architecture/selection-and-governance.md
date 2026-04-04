# Selection And Governance Contract

This document is the focused contract for fresh-planning behavior. If code differs from this contract, the code or this document must be updated.

## Ownership Model

- The server owns immutable facts such as lifecycle, trust, token, content size, and checksum metadata.
- The resolver owns policy, selection preferences, final candidate selection, dependency solving, governance, and lock generation.

## Policy And Preference Split

Hard policy decides what is legal.
Selection preferences decide what is preferred among legal candidates.

Current policy precedence:

1. per-request override
2. workspace policy
3. resolver defaults

Current selection-preference precedence:

1. CLI override
2. environment override
3. workspace config
4. user config
5. resolver default

## Candidate Policy Before Final Selection

Fresh planning requires a candidate-policy pass before final root selection.

This phase may reject candidates based on:

- lifecycle policy
- trust-tier policy
- token ceilings
- content-size ceilings
- other legality checks that do not require graph expansion

Final ranking must compare only policy-compliant candidates.

## Graph Governance After Resolution

Fresh planning also requires a post-resolution governance pass before lock generation.

This phase validates:

- full dependency graph legality
- aggregate policy limits when configured
- graph integrity before the lock becomes durable state

Governance failure after graph resolution must not silently fall through to another candidate unless the architecture is explicitly changed to permit that behavior.

## Interaction Rules

- Prompting is allowed only for root candidate ambiguity.
- Recursive dependency resolution must never prompt.
- `auto` may prompt when ambiguity remains and the session can prompt.
- `always` must fail clearly if prompting is required but unavailable.
- `never` must choose the top-ranked legal candidate deterministically.

## Fallback Rules

- missing `trust_tier` normalizes to `untrusted`
- missing `token_estimate` is `unknown`
- missing `content_size_bytes` is `unknown`

Unknown resource values:

- stay legal when no ceiling is configured
- fail closed when the corresponding ceiling is configured

## Checksum Rules

- phase 1 checksum algorithm is `sha256`
- the server publishes checksum facts
- the resolver verifies them during materialization
- checksum mismatch must fail fast as `ContentChecksumMismatchError`

## Lock And Explainability

The lock is the execution source of truth.

Explainability metadata may be stored with a lock when useful, but it must remain:

- separate from nodes, edges, and install order
- parse-preservable
- unnecessary for execution
