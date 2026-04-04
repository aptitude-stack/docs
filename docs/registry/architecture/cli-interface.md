# Aptitude CLI Interface Design

This document defines the current normative CLI interface shape for Aptitude.

It describes the human-facing command surface, the install-first wizard, the shared rendering and error rules, and the layering constraints that keep the CLI thin.

## Scope

This document covers:

- console entry and command routing
- public and advanced CLI surfaces
- wizard interaction design
- rendering, telemetry, and error behavior
- extension rules for future CLI changes

This document does not redefine resolver, governance, lockfile, or execution behavior. Those remain owned by the application and domain layers.

## Goals

The CLI exists to give humans one reliable local interface for:

- fresh planning from a natural-language query
- lock replay from an existing resolver lockfile
- reviewing the command surface without reading source code

The current UX goals are:

- wizard-first entry for casual and first-run use
- stable command signatures for repeatable terminal use
- review-first install flow before materialization
- human-readable failures without stack traces
- JSON output only when explicitly requested

## Non-Goals

The CLI must not:

- implement resolver or governance logic
- recompute ranking or legality decisions in the interface layer
- turn lock replay back into fresh planning
- expose raw internal exceptions as the default human experience

## Component Map

The CLI surface is split across five files:

- `src/aptitude_resolver/interfaces/cli/main.py`: root entrypoint; launches the wizard for zero-argument invocations and otherwise delegates to Typer
- `src/aptitude_resolver/interfaces/cli/app.py`: Typer command definitions and non-wizard command execution
- `src/aptitude_resolver/interfaces/cli/wizard.py`: guided install and sync flows, interactive prompts, and review panels
- `src/aptitude_resolver/interfaces/cli/catalog.py`: canonical command metadata, help text, manifest text, theme tokens, and separator conventions
- `src/aptitude_resolver/interfaces/cli/support.py`: shared workflow wiring, telemetry capture, TTY detection, option parsing, and user-facing error formatting

## Entry And Routing

The CLI currently routes requests with these rules:

- `aptitude` with no arguments launches the install-first wizard
- `aptitude install` with no query and no advanced overrides launches the install flow directly inside the wizard
- `aptitude sync` with no `--lock` and no `--json` launches the sync flow directly inside the wizard
- `aptitude install ...` with explicit arguments runs the Typer command path
- `aptitude sync ...` with explicit arguments runs the Typer command path
- `aptitude manifest` prints the full command and flag capability map
- hidden `aptitude resolve ...` exists for advanced preview and debugging of fresh planning without materialization

The CLI therefore supports both discovery-oriented use and automation-oriented use without maintaining two separate products.

For published one-off usage without a persistent install, the promoted entry examples are:

```bash
uvx aptitude-resolver@latest
uvx aptitude-resolver@latest install
uvx aptitude-resolver@latest install "<query text>"
uvx aptitude-resolver@latest sync
```

## Command Surface

### Public Commands

- `install`: fresh planning from a query plus local materialization
- `sync`: lock replay and local materialization from an existing lockfile
- `manifest`: human-readable capability map for commands and flags

### Advanced/Internal Command

- `resolve`: hidden preview/debug surface that follows the fresh-planning path but stops before materialization and prints stable JSON

### Global Framework Flags

- `--version`: print the installed Aptitude version and exit
- `--help`: show command help
- `--install-completion`: install shell completion for the current shell
- `--show-completion`: print shell completion for manual installation or customization

## Catalog-Driven Help Contract

`catalog.py` is the canonical metadata source for:

- root help text
- command help text
- manifest output
- wizard-facing capability summaries
- shared CLI visual tokens

When a command, example, or flag changes, the catalog must be updated in the same change. The command definitions in `app.py` and the help/manifest content in `catalog.py` are intentionally separate, but they must stay aligned.

## Wizard Design

The wizard is the default human entrypoint. It is install-first, review-first, and intentionally narrower than the full command surface.

### Launcher

The zero-argument wizard launcher renders:

- the Aptitude wordmark
- a one-line product description
- a fixed-width separator
- a flow chooser with `install`, `sync`, `help`, and `exit`

The launcher exists to reduce cold-start friction while still exposing the capability map on demand.

### Install Flow

The guided install flow is:

```text
query input
-> selection profile
-> interaction mode
-> resolve
-> optional candidate selection
-> review plan
-> confirm
-> install
-> installed skills panel
```

Key properties:

- the query prompt uses a larger free-text input surface
- selection profile is explicit: `balanced`, `low-cost`, or `high-trust`
- interaction mode is explicit: `auto`, `always`, or `never`
- ambiguity remains root-only; the wizard may ask the user to choose one candidate
- the user sees a compact review panel before installation begins
- cancellation before installation is explicit and non-destructive

The install flow also supports returning from profile selection back to the query prompt. That is the current escape hatch for revising intent without restarting the whole wizard.

### Sync Flow

The guided sync flow is intentionally smaller:

```text
lockfile path
-> target directory
-> sync
-> installed skills panel
```

Sync must remain lock-driven. The wizard must not introduce discovery, reranking, or re-resolution into this path.

### Help Flow

The wizard help branch renders the compact capability map panel from `catalog.py`, then returns the user to the launcher instead of exiting the session.

## Interaction Model

The CLI uses different interaction surfaces depending on capabilities:

- if both stdin and stdout are TTYs, the wizard uses richer interactive prompts
- if `prompt_toolkit` is unavailable, the wizard falls back to simpler terminal input and raw keyboard menus
- if stdin or stdout is not a TTY, text prompts fall back to plain `input(...)` style prompts

Current keyboard rules:

- launcher and menus use arrow-key navigation
- `enter` confirms a menu selection
- `q` cancels menu selection where supported
- the large install-query text box uses `Ctrl+D` to submit
- `Ctrl+C` cancels interactive prompt-toolkit flows

The CLI should prefer the simplest interaction model that works in the current terminal rather than assuming a fully featured environment.

## Rendering Rules

The current CLI rendering contract is:

- the wizard uses a shared 140-character separator line
- human-facing summaries use Rich panels, text styling, and transient status/progress indicators
- help and manifest output stay text-first and copy-paste-friendly
- install and sync success summaries are concise and package-manager-like
- the wizard should feel structured, but it must remain readable in ordinary terminals without relying on full-screen layouts

The interface is an interactive CLI wizard, not a separate TUI product. Future changes should preserve that mental model unless the docs are updated explicitly.

## Telemetry And Activity Feedback

Telemetry is additive and folded into the CLI instead of streamed as raw logs.

Current behavior:

- command and wizard flows execute inside `capture_cli_telemetry()`
- human-facing command runs render folded stage timings after execution
- interactive command runs render status spinners or transient progress bars during long operations
- telemetry must not change correctness, ordering, or decision-making

This preserves observability without leaking internal mechanics into the main UX.

## Error And Exit Behavior

The CLI distinguishes between expected resolver failures, configuration problems, and unexpected internal failures.

### Expected Human-Facing Errors

`support.py` renders tailored messages for:

- missing environment configuration
- invalid CLI or workspace configuration
- invalid explicit candidate selection
- invalid lockfiles
- checksum mismatches
- no candidate results
- policy violations

These messages must stay specific, actionable, and free of traceback noise.

### Exit Semantics

- Typer command validation failures for missing required user input exit with code `2`
- resolver and sync failures exit with code `1`
- unexpected exceptions are converted into a sanitized internal-error message and exit with code `1`
- wizard cancellations render `Cancelled.` and return without a traceback

## Layering And Ownership Rules

The CLI is a presentation layer over `InstallWorkflowService`.

It may:

- gather human input
- build validated workflow options
- choose whether prompting is possible
- render progress, summaries, panels, and errors
- label selection source for traceability

It must not:

- select candidates on its own
- reinterpret policy results
- resolve dependencies outside the application layer
- alter lock replay semantics

The interface owns presentation.
The application owns orchestration.
The resolver owns decisions.

## Current Invariants

Any CLI change must preserve these current invariants:

- zero-argument entry launches the wizard
- `install` remains the primary human fresh-planning entrypoint
- `sync` remains strictly lock-driven
- `resolve` remains the preview/debug surface, not the public default
- JSON output is opt-in and stable for machine use
- interactive ambiguity handling stays root-only
- help and manifest output remain catalog-driven
- CLI telemetry, cache, and retries remain additive and must not alter correctness

## Extension Rules

When extending the CLI:

1. update `catalog.py` when commands, examples, or flags change
2. keep wizard flows narrower and more opinionated than raw command usage
3. preserve non-interactive fallbacks for automation and CI
4. prefer adding presentation logic in `wizard.py` or `app.py`, not business logic
5. update canonical docs in the same change when behavior or routing changes

Good extensions:

- new rendering panels
- better review surfaces
- clearer help and manifest descriptions
- additional safe prompting around existing application capabilities

Bad extensions:

- interface-only policy rules
- reimplementing resolver decisions in the wizard
- making `sync` depend on discovery data
- introducing incompatible command signature drift without explicit migration

## Verification Expectations

Relevant verification currently lives in:

- `tests/unit/interfaces/cli/test_app.py`
- `tests/unit/interfaces/cli/test_wizard.py`
- `tests/unit/interfaces/cli/test_help_surface.py`
- `tests/unit/interfaces/cli/test_support.py`

Changes to routing, prompts, help, or rendering should update those tests in the same change.
