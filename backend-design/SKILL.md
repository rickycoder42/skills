---
name: backend-design
description: Architecture and production-readiness guidance for server-side code — HTTP/GraphQL/RPC APIs, services, background workers, queue consumers, scheduled jobs, WebSocket servers, CLIs. Framework-, runtime-, and database-agnostic. Use when building, extending, reviewing, or designing any backend system or API.
---

# Backend Design

Guidance for building backend code that is correct, secure, and operable in production. It is framework-, runtime-, and database-agnostic: apply the principles using whatever the project already uses. Also apply `code-structure` (organization) and, for TypeScript, `typescript-style` (language conventions). Established repo conventions take precedence.

## 1. Understand Before Building

Clarify these before writing code — from the codebase first, then from the user if still unclear:

- **Domain**: core entities, invariants, and the operations on them.
- **Boundaries**: what this service owns, what it calls, who calls it, and through which contracts.
- **Load and data**: expected traffic, data volume, latency needs, consistency needs.
- **Constraints**: existing stack (runtime, framework, database, queue, deployment target), auth model, compliance.

Choose the **simplest architecture that fits** and apply it consistently. A well-structured modular monolith is the right default; introduce queues, caches, or separate services only when a concrete requirement demands them.

Stack choice: use what the repo already uses. For a new project, pick based on requirements and state the choice and reasoning; ask if the user is likely to have a preference.

## 2. Layering

Keep three responsibilities separate, whatever the folders are called:

- **Transport / entry layer** (routes, controllers, resolvers, message handlers, CLI commands, cron entry points): parse and validate input, authenticate, call the application layer, map the result or error to the transport's response. No business rules, no queries.
- **Application / domain layer** (services, use cases): business rules and orchestration. Transport-agnostic — the same service is callable from HTTP, a queue consumer, a test, or a script. Implemented as classes with dependencies injected through the constructor.
- **Infrastructure layer** (repositories, gateways, adapters for DB, cache, queue, email, third-party APIs): all I/O. Hidden behind small interfaces the domain depends on.

Wire dependencies in one composition root at startup. Do not import singletons (DB clients, loggers, config) directly inside services.

## 3. API Contract

- Consistent resource naming and casing across the whole API — pick one convention and use it everywhere.
- One response shape for success and one for errors across all endpoints.
- Correct status codes (or the protocol's equivalent): `400` invalid input, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict, `422` semantic validation failure (if the repo uses it), `429` rate-limited, `5xx` only for server faults.
- Errors carry a stable machine-readable `code`, a human-readable `message`, and optional field-level `details`. Consider RFC 9457 (Problem Details) for HTTP APIs.
- Lists are always paginated and bounded (max page size). Prefer cursor pagination for large or frequently changing datasets.
- Explicit, versioned contract once there are external consumers; prefer additive, backward-compatible changes.
- If the API is described by a schema (OpenAPI, GraphQL SDL, protobuf), keep the schema as the source of truth and generate types/clients from it.

## 4. Input Validation

- Validate **every** external input at the boundary: body, query, path params, headers, cookies, messages from queues, webhook payloads, env vars, files.
- Use schema validation (whichever library the repo uses) and derive types from schemas.
- Reject unknown or excessive input: size limits, length limits, array limits.
- After validation, pass typed data inward; the domain does not re-validate the same shape (it still enforces business invariants).

## 5. Error Handling

- Distinguish **expected domain errors** (not found, conflict, validation, permission) from **unexpected failures** (bugs, infrastructure down).
- Domain errors are explicit types thrown/returned by services; one central error mapper translates them to transport responses.
- Unexpected errors are logged with full context (and `cause` chain) and returned to clients as a generic `500` with a correlation ID — never stack traces, SQL, or internal identifiers.
- Never swallow errors. Every `catch` either handles the error meaningfully or rethrows/propagates with added context.

## 6. Security

- **Authentication** at the edge; **authorization** checked for every operation on every resource (object-level checks — "can _this_ user access _this_ record?"), not only per route.
- Parameterized queries / query builder only — never string-concatenated queries or commands.
- Secrets only from environment or a secret manager; never in code, logs, error messages, or responses.
- Hash passwords with a modern slow KDF (argon2id, scrypt, bcrypt). Compare secrets in constant time.
- Rate-limit authentication and expensive endpoints. Configure CORS, security headers, and cookie flags (`HttpOnly`, `Secure`, `SameSite`) deliberately.
- Do not leak existence of resources or accounts through different error messages or timings when that matters.
- Treat uploaded files, URLs supplied by users (SSRF), and deserialized data as hostile.

## 7. Configuration

- One config module loads and validates all configuration at startup; the process fails fast with a clear message if anything required is missing or invalid.
- The rest of the code receives a typed config object; no `process.env`/equivalent reads scattered through the code.
- Same code in every environment — behaviour differences come from config, not from `if (env === "production")` branches in business logic.

## 8. Data and Persistence

- Schema changes only via versioned migrations; never hand-edit schemas. Migrations must be safe to run against live data (expand → migrate → contract for breaking changes).
- Data access only in the infrastructure layer.
- Use transactions for operations that must succeed or fail together; keep them short and never perform network calls to third parties inside them.
- Enforce invariants in the database too (constraints, unique indexes, foreign keys), not only in application code.
- Avoid N+1 queries; select only needed columns; add indexes for the queries you actually run.
- Configure connection pooling for the deployment model (long-running process vs. serverless).

## 9. Reliability

- Every outbound call (HTTP, DB, cache, queue) has a timeout. Propagate cancellation where the platform supports it.
- Retry only idempotent operations, with exponential backoff and jitter, and a bounded number of attempts.
- Make writes that clients or queues may repeat **idempotent** (idempotency keys, upserts, deduplication by message ID).
- Queue consumers and jobs: at-least-once delivery is the norm — handle duplicates, use dead-letter handling for poison messages, and make jobs resumable.
- Graceful shutdown: on termination signals stop accepting new work, finish in-flight work within a deadline, close connections.
- Do not keep request-scoped state in process memory if the service runs more than one instance.

## 10. Observability

- Structured logs (JSON) with consistent fields: timestamp, level, message, request/correlation ID, and relevant domain IDs. Propagate the correlation ID across service and queue boundaries.
- Meaningful levels: `error` needs attention, `warn` is unusual but handled, `info` is a business-relevant event, `debug` is for development.
- Never log secrets, tokens, passwords, or unnecessary personal data.
- Liveness and readiness endpoints for deployed services (readiness checks dependencies).
- Expose metrics/traces if the platform has them (latency, error rate, throughput, queue depth).

## 11. Testing

- Domain/application logic: unit tests with injected fakes — enabled by constructor injection.
- Infrastructure: integration tests against real dependencies (containerized DB, etc.) where practical.
- API: a few contract/end-to-end tests over the real transport for the main flows and error shapes.
- Test the failure paths (validation errors, authorization denials, conflicts, timeouts), not only the happy path.

## Anti-Patterns

- **God handlers** — one handler that validates, queries, applies rules, sends emails, and formats the response.
- **Business logic in middleware** or in database triggers nobody knows about.
- **Trusting input** — using request data before validation; authorization only at the route level.
- **Silent failures** — empty `catch`, generic `500` with no log, fire-and-forget without error handling.
- **Unbounded anything** — queries without limits, lists without pagination, calls without timeouts, retries without caps.
- **Inconsistent contracts** — mixed casing, different envelopes, ad-hoc error formats.
- **Over-engineering** — microservices, event buses, CQRS, or generic repositories for a simple CRUD; abstractions before the second real use case.
- **Hardcoded config and secrets.**

## Before Finishing

- [ ] Entry points are thin; business rules live in services; I/O lives in adapters.
- [ ] All external input validated; authorization checked per resource.
- [ ] Error responses are consistent and leak nothing internal.
- [ ] Config validated at startup; no secrets in code or logs.
- [ ] Outbound calls have timeouts; retried/replayed operations are idempotent.
- [ ] Migrations included for schema changes.
- [ ] Tests cover main flows and failure paths.
