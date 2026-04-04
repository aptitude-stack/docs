# Aptitude PRD: Resolver Subsystem

## 1. What is this part of the product?

The Aptitude Resolver is the local decision-making subsystem of Aptitude. It takes either a user or agent request for a capability, or an existing lockfile, and turns that input into a deterministic plan for which skills should be used locally and how they should be materialized.

Within the broader Aptitude system, this subsystem is responsible for decisions, not storage. The registry and publisher side of Aptitude provide immutable facts, metadata, and artifacts. The resolver side interprets intent, selects candidates, resolves dependencies, applies governance rules, produces a lock, and drives local execution. In simple terms: the server knows what exists, and the resolver decides what should actually be used.

## 2. What problems does it help solve?

### Accessibility for humans and agents

Managing AI skills becomes difficult when the only interface is either too low-level for people or too ambiguous for automation. This subsystem addresses both sides of that problem.

For humans, it provides a guided install-first experience and a simpler command surface. For agents and automation, it provides deterministic commands, lock replay, JSON output, and a stable manifest of available capabilities. That matters because AI systems need interfaces they can call reliably without depending on guesswork or undocumented behavior.

### Quality and security of skills

Not every available skill should be installed or used. Skills can vary by trust level, lifecycle status, content size, and other governance-relevant attributes. This subsystem helps ensure that selection is not just convenient, but safe and policy-aware.

It does this by separating "what is available" from "what is allowed" and "what should be chosen." Candidates are filtered through policy checks, the resolved dependency graph is validated before it becomes durable state, and downloaded content is verified against published checksums. This reduces the risk of low-trust, deprecated, oversized, or tampered content entering the local environment.

### Governance and controlled usage

Aptitude needs a way to enforce usage rules without making every user manually inspect every dependency. This subsystem contributes that control layer.

It supports workspace-level policy, stricter per-run controls for fresh planning, and a lock-based flow that preserves approved decisions once they have been made. This helps teams move from ad hoc skill installation toward repeatable and governable usage.

### Dependency management and modularity

AI skills are only useful as a system if their dependencies can be selected consistently and replayed later. This subsystem solves the package-manager-style problem for skills: picking compatible versions, expanding transitive dependencies, validating the graph, and turning the result into a lock that can be executed again later.

That matters because modular skills are only practical when the complexity of dependency resolution is handled centrally and deterministically rather than by each user or agent.

## 3. What solution does it provide?

This subsystem provides a local resolver and execution-planning layer for AI skills.

Its core solution is a two-path model:

- Fresh planning: start from a natural-language request, discover viable candidates, choose the best legal option, resolve its dependencies, validate the result, and produce a lock-backed installation plan.
- Lock replay: start from an existing lockfile and reproduce the same installation outcome without re-running discovery or dependency solving.

This design gives Aptitude both flexibility and control. Users can start from intent when they need something new, and agents can rely on locked, replayable results when they need consistency. The subsystem therefore acts as the bridge between a broad skill ecosystem and a safe, reproducible local working state.

## 4. Who uses this?

### End users

Developers, operators, and technically inclined users interact with this subsystem when they want to install skills from a request, inspect the available command surface, or replay a known-good skill set from a lockfile.

### Governance and platform users

Workspace owners and platform stewards rely on this subsystem to enforce policy, limit what can be selected, and make local skill usage auditable and repeatable.

### AI agents and automation

Agents use this subsystem as a system consumer when they need a stable way to request capabilities, inspect supported options, consume machine-readable output, and replay prior decisions without introducing fresh ambiguity.

## 5. What is the experience?

In practice, this subsystem participates in two main flows.

### Fresh planning flow

The user or agent starts with a capability request. The subsystem accepts a query, optional selection preferences, and optional governance constraints. It then produces a reviewed plan: which root skill was selected, which dependencies are required, whether the result is policy-compliant, and what will be materialized locally. For humans, this can happen through a guided CLI flow. For automation, it can happen through stable command flags and JSON output.

### Lock replay flow

The user or agent starts with an existing lockfile. The subsystem reads the lock, reconstructs the execution plan, verifies integrity requirements, and materializes the locked skill set locally. This path is intentionally narrower because the goal is not to decide again, but to reproduce an approved result.

### Inputs and outputs

Inputs include natural-language queries, lockfiles, workspace configuration, selection preferences, and governance constraints. Outputs include a resolved plan, a durable lockfile, local materialized skills, machine-readable results for automation, and human-readable summaries for interactive use.

## 6. Features and why they exist

### 1. Query-based skill planning

What it does: lets users start from intent rather than exact skill identifiers.

Why it exists: most users and agents know the capability they want, not the exact package or version they need.

How it contributes: it makes the ecosystem discoverable and usable without exposing the full complexity of the underlying registry.

### 2. Deterministic candidate selection and dependency resolution

What it does: chooses a final root candidate, selects compatible versions, and resolves the full dependency graph.

Why it exists: modular skills are only practical if transitive dependencies can be assembled consistently and repeatably.

How it contributes: it turns a broad set of available options into one concrete, reproducible solution.

### 3. Policy-aware governance before locking

What it does: applies rules around trust, lifecycle status, size, token ceilings, and graph legality before a lock is accepted.

Why it exists: availability alone is not enough; Aptitude needs to prevent unsafe or non-compliant skill sets from being installed.

How it contributes: it provides controlled usage and creates a boundary between what the registry exposes and what a workspace is actually allowed to consume.

### 4. Lockfile generation and replay

What it does: records the resolved outcome as a durable artifact and enables later replay through `sync`.

Why it exists: users and agents need a stable execution source of truth after a decision has been made.

How it contributes: it makes installations reproducible, reviewable, and suitable for both local workflows and automation.

### 5. Local execution planning and materialization

What it does: turns a resolved or replayed lock into an installation plan and materializes the required skill content locally.

Why it exists: planning only matters if it leads to usable local capability.

How it contributes: it closes the loop from selection to actual availability in the working environment.

### 6. Integrity verification

What it does: verifies downloaded content against published checksum facts.

Why it exists: users need confidence that installed content matches what was approved and published.

How it contributes: it strengthens the trust model of the whole system and helps prevent silent corruption or tampering.

### 7. Human-and-agent accessible interface

What it does: offers a guided CLI wizard for interactive use, stable commands for scripting, manifest output for capability discovery, and JSON output for automation.

Why it exists: Aptitude must be usable by both people and AI systems without maintaining separate products.

How it contributes: it makes the resolver the practical access point for local skill planning and replay today, even before broader SDK or MCP surfaces exist.

### 8. Telemetry and traceability

What it does: captures stage-level timing and trace information across discovery, selection, resolution, locking, and execution.

Why it exists: users and platform teams need observability into why work took time and how decisions were reached.

How it contributes: it improves trust, debugging, and operational confidence without changing decision correctness.

## 7. How it fits in the system

This subsystem depends on Aptitude's registry/server side for immutable metadata, indexed candidate retrieval, published artifacts, and checksum facts. It also depends on local workspace configuration and the local environment where skills will be materialized.

Other parts of the system depend on it as the decision and replay engine. The CLI in this repo is its current primary interface. Future agent-facing surfaces such as SDK or MCP are implied consumers but are not yet implemented here.

Its relationship to adjacent Aptitude subsystems is straightforward:

- Registry/server: provides facts and artifacts that the resolver consumes.
- Publisher: creates and publishes the skills and immutable metadata that eventually become resolver inputs through the registry.
- Resolver/client: decides what to use, validates it, locks it, and makes it available locally.

## 8. Gaps and future direction

Several next-step capabilities are implied by the repo but not yet fully realized.

- Organization-managed policy sources appear to be a planned direction, but governance today is still centered on defaults, workspace config, and stricter per-run overrides.
- MCP and SDK interfaces are explicitly deferred, which means agent accessibility currently relies mostly on the CLI and machine-readable CLI output rather than a dedicated programmatic integration layer.
- Plugins and extensibility are referenced as future work, suggesting the system may eventually support broader ecosystem integration beyond the current built-in flows.
- Explanation quality appears to be improving but not yet complete, especially around clearly explaining why one candidate won over another.
- Governance is present and meaningful today, but broader policy sophistication and organizational controls are likely future investments.

These gaps suggest the current subsystem is already strong in deterministic local resolution and replay, while the next phase will likely focus on richer governance, better explanation, and more direct agent-consumable interfaces.

## 9. Summary

The Aptitude Resolver is the subsystem that turns available AI skills into approved, reproducible local capability. It is the part of Aptitude that interprets intent, chooses what to use, resolves dependencies, applies governance, creates a lock, and drives local materialization.

It matters because Aptitude is not just a registry of skills. The system only becomes usable when someone or something can safely decide what to install, reproduce that decision later, and make the result accessible to both humans and agents. This subsystem is the decision engine that makes that possible.
