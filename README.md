# AI Agent Security Gateway

A policy-enforcing gateway that controls and audits how AI agents access business tools.

> Status: design foundation, version 0.1, 25 November 2026. This repository contains planning documents only. No application, contracts, integrations or deployment are implemented yet. Proposed choices and targets require validation.

## Product purpose

Agents can invoke sensitive tools with broad credentials, while application logs often fail to explain whose authority was used. Enterprises need enforceable boundaries at the point of tool execution.

## Intended customer

Platform and security teams deploying tool-using agents; initial buyer: the team responsible for production agent access.

## First commercial scope

An HTTP gateway for a small set of tool APIs, with per-agent identities, deterministic allow/deny policies, approval workflows and attributable audit events.

## Planned capabilities

- Agent registration, tenant-scoped identities and short-lived credentials.
- Versioned policies covering tool, action, resource, tenant, rate and spend constraints.
- Allow, deny and require-approval decisions with structured reason codes.
- Connector adapters that validate schemas and restrict destinations and operations.
- Request deduplication, quotas, credential revocation and emergency disable.
- Audit dashboard and export with secret and personal-data redaction.

## Proposed implementation

Proposed: TypeScript service and SDK, React administration UI, PostgreSQL, an established policy engine or a deliberately small typed policy evaluator, OpenTelemetry and an external secret manager. Choose MCP integration only after the initial HTTP contract works.

These are initial design proposals, not installed dependencies or compatibility claims. Select and pin versions during implementation and record significant changes as architecture decisions.

## Documentation

- [Design brief](docs/DESIGN_BRIEF.md): product scope, architecture, domain model, invariants, evaluation, delivery phases and open decisions.

## Repository contents

- `README.md`: project overview and entry point.
- `docs/DESIGN_BRIEF.md`: expandable engineering and product brief.
- `.gitignore`: excludes credentials, build output and local runtime files.

Future application directories are proposed in the design brief; they do not exist yet. There is no runnable setup command at this stage.

## First implementation task

Implement two mock tools, signed test identities, explicit allow/deny rules and request-digest approval binding; demonstrate denied tenant crossover.

## Commercial hypothesis

Subscription per organisation or environment plus metered protected executions. Initial value proposition: attributable tool access and enforceable policies for existing agents.

Pricing, customer demand and production readiness are unvalidated. The initial deliverable is a narrow, measurable vertical slice.

## Development approach

1. Resolve the first-scope decisions in the design brief.
2. Define deterministic contracts and independent acceptance fixtures.
3. Implement the smallest complete workflow with simulated integrations.
4. Add real integrations, recovery behaviour and measured evaluations.
5. Review pilot evidence before expanding scope.

## License

No open-source license has been selected. Adding one is a separate owner decision; repository visibility does not itself grant a license.
