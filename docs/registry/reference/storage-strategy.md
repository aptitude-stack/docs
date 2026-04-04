# Storage Strategy

This document describes how aptitude stores and materializes local state today.

## Current Strategy

- registry data is read remotely through `registry/`
- immutable content may be cached locally as an advisory optimization
- lockfiles are the durable local execution input
- materialization writes the selected skills into the requested target workspace

## Constraints

- caches must never become a hidden source of truth
- lock replay must remain valid without discovery or dependency solving
- checksum verification happens during materialization, not as an afterthought

## Deferred Areas

- richer cache invalidation strategy beyond current advisory behavior
- broader local storage abstractions if plugins, SDK, or MCP surfaces require them
