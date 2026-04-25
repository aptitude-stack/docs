# Interface Expansion

This document tracks interface surfaces that are expected but not yet implemented.

## Planned Interfaces

- `interfaces/sdk/`: programmatic consumer surface over resolver workflows
- `interfaces/mcp/`: MCP-facing resolver surface for model-driven integrations
- `plugins/`: extensibility model for future resolver capabilities

## Constraints

Any new interface must preserve the existing invariants:

- fresh planning still flows through discovery, resolver, governance, and lock generation
- lock replay stays lock-driven
- interfaces remain thin and do not bypass application orchestration
- telemetry and explainability remain additive
