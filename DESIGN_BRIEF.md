# Design brief: AI Agent Security Gateway

Version: 0.1  
Date: 25 November 2026  
Status: proposal; no implementation exists.  
Repository: `agent-security-gateway`

## 1. Problem and outcome

Agents can invoke sensitive tools with broad credentials, while application logs often fail to explain whose authority was used. Enterprises need enforceable boundaries at the point of tool execution.

Desired outcome: A policy-enforcing gateway that controls and audits how AI agents access business tools.

## 2. Customer and commercial hypothesis

Platform and security teams deploying tool-using agents; initial buyer: the team responsible for production agent access.

Subscription per organisation or environment plus metered protected executions. Initial value proposition: attributable tool access and enforceable policies for existing agents.

Before a paid pilot, interview prospective buyers, inspect their current workflow, establish a baseline and agree on measurable success criteria. Demand and willingness to pay are hypotheses, not established facts.

## 3. MVP boundary

An HTTP gateway for a small set of tool APIs, with per-agent identities, deterministic allow/deny policies, approval workflows and attributable audit events.

### In scope

- Agent registration, tenant-scoped identities and short-lived credentials.
- Versioned policies covering tool, action, resource, tenant, rate and spend constraints.
- Allow, deny and require-approval decisions with structured reason codes.
- Connector adapters that validate schemas and restrict destinations and operations.
- Request deduplication, quotas, credential revocation and emergency disable.
- Audit dashboard and export with secret and personal-data redaction.

### Out of scope

Claiming to solve all prompt injection, replacing enterprise identity providers, arbitrary transparent network interception, or using an LLM as the final access-control authority.

## 4. Primary user journey

An administrator registers an agent, assigns tool scopes and configures policies. An agent submits a tool request using a short-lived identity. The gateway validates the request and policy, obtains approval when necessary, injects a scoped upstream credential, executes the permitted request and records a redacted audit event.

The interface must expose current status, evidence, unresolved exceptions and the next available action. Implementation details belong in operator views unless they affect a user decision.

## 5. Architecture proposal

Administration UI and API; identity validator; deterministic policy evaluator; approval service; connector executor; secret-manager integration; append-only audit pipeline; PostgreSQL. Keep the enforcement path small and separate from asynchronous analytics.

### Technology direction

Proposed: TypeScript service and SDK, React administration UI, PostgreSQL, an established policy engine or a deliberately small typed policy evaluator, OpenTelemetry and an external secret manager. Choose MCP integration only after the initial HTTP contract works.

Start as a modular application with separately isolated workers where required. Keep third-party providers behind typed interfaces. Avoid premature microservices. Record selected dependency versions, deployment assumptions and rejected alternatives when implementation starts.

### Suggested future repository layout

- `apps/web/`: user and administration interfaces.
- `services/api/`: authenticated product API and domain coordination.
- `services/worker/`: durable asynchronous execution and recovery.
- `packages/contracts/`: shared schemas and integration contracts; blockchain projects should keep Solidity separately under `contracts/`.
- `tests/fixtures/`: synthetic or explicitly permitted evaluation data.
- `tests/integration/`: state transitions, adapter failures and recovery scenarios.
- `docs/decisions/`: numbered architecture decisions.
- `infra/`: deployment definitions after the runtime is chosen.

This is a proposed layout, not a scaffold already present.

## 6. Domain model and lifecycle

Core entities: Tenant, AgentIdentity, CredentialBinding, ToolDefinition, PolicyVersion, AuthorizationDecision, Approval, Execution, AuditEvent.

received → authenticated → evaluated → approved/denied/pending_approval → executing → succeeded/failed/unknown. An approval expires and binds to a request digest, resource, policy version and actor.

All persisted business records carry tenant scope, stable IDs, creation timestamps and a version where concurrent updates are possible. Record external IDs separately from internal IDs. Use database constraints for uniqueness and transaction boundaries for state changes. Persist event/outbox records in the same transaction as the associated business transition.

## 7. API and integration contracts

POST /v1/agents; POST /v1/policies; POST /v1/tool-executions; POST /v1/approvals/{id}/decision; POST /v1/agents/{id}/revoke; GET /v1/audit-events. The execution endpoint returns a decision ID and a stable execution ID.

Define request/response schemas and error codes before adapter implementation. Store request digests alongside idempotency keys; reject reuse with different payloads. Bound retries and distinguish safe replay from an unknown external side effect. Sign outbound webhooks and support receiver deduplication and replay windows.

## 8. Correctness and security invariants

- Unknown tools, invalid schemas and missing identities are denied by default.
- Only the gateway holds upstream secrets; model-visible responses never contain raw credentials.
- Approval cannot be reused after arguments, destination, tenant or policy context change.
- Revocation is checked at execution time, including after an approval wait.
- A timeout after an upstream side effect is recorded as unknown, not blindly retried.
- A model-generated risk signal may tighten or flag a decision but cannot override a deterministic denial.

Authenticate people and services separately. Apply role-based access and tenant isolation at every read/write boundary, including artifacts and background jobs. Keep secrets in an approved secret manager, redact logs and define rotation/revocation procedures. Treat uploaded documents, repository content, RPC responses and model output as untrusted at their respective boundaries.

## 9. Data, AI and evaluation strategy

Build a labelled adversarial request corpus covering tenant crossover, argument tampering, SSRF destinations, token expiry, approval replay, secret leakage and burst traffic. Measure unauthorized-action prevention, legitimate-task completion, redaction failures, false denials and added p50/p95 latency. Assess semantic injection detection separately from hard authorization guarantees.

Use synthetic or explicitly authorized data initially. Version fixtures, evaluator code, prompts and model configuration where applicable. Report denominators, failed cases, latency and cost together; do not publish only a success percentage. Model-based judgments may supplement but cannot replace deterministic domain checks. Blockchain components do not require an AI feature unless a measured use case justifies one.

## 10. Reliability and operations

Assume jobs can be delivered more than once and workers can crash between any two external calls. Persist checkpoints and reconcile uncertain outcomes. Provide bounded retry policies, a dead-letter/exception queue and an operator procedure for manual recovery. Cancellation and timeout behaviour must be specified for each external adapter.

Capture structured logs, correlation IDs, traces, queue age, failure counts, integration latency and per-tenant usage. Never log raw secrets or confidential payloads by default. Add health/readiness endpoints when services exist. Define backup retention, restore procedures and a documented recovery exercise before a pilot.

Performance and availability targets remain open until a representative workload and deployment budget are agreed. Baseline p50/p95 latency, throughput, storage growth and unit cost during the vertical slice; record proposed targets separately from measured results.

## 11. Delivery plan and phase exits

### Phase 1

Foundation: threat model, identity and tool schemas, deterministic policies and adversarial fixtures.

Exit: demonstrate this phase’s workflow with reproducible evidence and document unresolved failures before proceeding.

### Phase 2

Vertical slice: one mock read tool and one mock write tool through the enforcement path.

Exit: demonstrate this phase’s workflow with reproducible evidence and document unresolved failures before proceeding.

### Phase 3

Integration: real connector, external secret manager, approval UI and audit export.

Exit: demonstrate this phase’s workflow with reproducible evidence and document unresolved failures before proceeding.

### Phase 4

Pilot: revocation and failure drills, load measurements, SDK integration and operator runbooks.

Exit: demonstrate this phase’s workflow with reproducible evidence and document unresolved failures before proceeding.

## 12. Acceptance scenarios

- Agent A cannot execute a tool or access a resource granted only to agent B or another tenant.
- Changing an approved request invalidates its approval, and expired/revoked identities cannot execute.
- Secrets do not appear in responses, traces, errors or exported audit records.
- Gateway or policy-service failure denies protected operations with an explicit error.
- An upstream timeout creates an unknown result; a duplicate request cannot cause a second unapproved side effect.

These are planned checks, not passing test results. Each scenario should become an independent integration or system test where practical. Include restart, duplicate-event and authorization cases before a paid pilot.

## 13. Risks and dependencies

Gateway bypass, overly broad upstream credentials, policy complexity, audit-data exposure and upstream tools without idempotency. The deployment topology must make the gateway the actual route to protected credentials.

Maintain a risk register with an owner, mitigation and evidence. Production use depends on the final operating model, provider contracts, deployment controls and independent review appropriate to the system; this brief makes no compliance or production-readiness claim.

## 14. Open decisions

- Which identity provider and token exchange define the first deployment?
- Which upstream connector offers the best initial customer value?
- Must audit events be externally signed or exported to customer storage?
- What latency and availability objectives can the first deployment support?

Assign decision owners and dates when the project begins. Capture consequential decisions in `docs/decisions/`; update this brief when scope changes.

## 15. Immediate next task

Implement two mock tools, signed test identities, explicit allow/deny rules and request-digest approval binding; demonstrate denied tenant crossover.

Deliver a small reviewable change, fixture-based evidence and a README update describing what actually runs. Do not mark future features as implemented.

## 16. Expansion backlog

- Customer discovery notes and agreed pilot success metrics.
- Detailed component diagrams and trust boundaries.
- Versioned API schemas and example payloads.
- Entity relationships, migrations and retention rules.
- Threat model and integration-specific failure matrix.
- Evaluation report, operating cost model and support assumptions.
- Deployment runbook, incident procedure and recovery evidence.

## 17. Starting references

The following are starting documentation links, not evidence that a particular dependency version has been selected or audited. Verify applicable documentation during implementation.

- https://modelcontextprotocol.io/
- https://opentelemetry.io/docs/
- https://www.openpolicyagent.org/docs/
