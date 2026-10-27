# Documentation Index

| Document | Purpose |
|---|---|
| [Low-Level Design](LLD.md) | Complete architecture, domain model, diagrams, data design, workflows, security, testing, operations, risks |
| [API Reference](API.md) | Public and internal endpoints, DTOs, authorization, status codes, examples |
| [Testing Guide](TESTING.md) | Test strategy, suites, isolation model, commands, coverage matrix |
| [Project README](../README.md) | Local setup, manual execution, end-to-end checks, troubleshooting |

## Suggested Reading Order

1. `README.md` to run the system.
2. `LLD.md` sections 1–5 for context and architecture.
3. `LLD.md` sections 6–9 for service and data design.
4. `LLD.md` sections 10–14 for workflows, security, testing, and operations.
5. `API.md` while integrating a client.
6. `TESTING.md` before changing behavior.

## Diagram Conventions

All diagrams are written in Mermaid and render in GitHub, GitLab, VS Code, and most Markdown viewers. The diagrams are logical views. Physical service boundaries and database ownership are described in the LLD.

## Documentation Scope

The documentation covers the implemented backend only. It does not define a frontend, payment processing, housekeeping workflows, reporting, or production cloud infrastructure. Those items are listed as future work in the LLD.
