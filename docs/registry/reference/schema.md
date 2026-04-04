# Resolver Schema Surfaces

The resolver does not own a database schema in this repository. Its canonical local schema surfaces are configuration and lock data.

## Configuration Surface

Current configuration sources:

- environment variables
- workspace `aptitude.toml`
- optional user-level `aptitude.toml`

These sources shape policy and selection preferences, but they do not replace resolver defaults unless the precedence rules permit it.

## Lockfile Surface

The lockfile is the durable resolved schema for execution.

It must preserve:

- selected coordinates
- graph shape
- install order
- governance outcomes
- minimal policy snapshot
- optional explainability metadata that remains non-executable

## Why This File Exists

The registry docs use `reference/schema.md` for canonical persistence shape. The resolver analogue is the set of local schema surfaces it owns directly: config and lock data.
