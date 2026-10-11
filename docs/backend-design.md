# Backend design

[← 中文首页](../README.md) · [English home](../README_EN.md)

## Architectural style

The FastAPI backend uses a layered structure:

```text
HTTP request
    │
    ▼
Router ───── schemas and dependency checks
    │
    ▼
Service ──── business workflow and transactions
    │
    ▼
Repository ─ data-access queries
    │
    ▼
SQLAlchemy model + PostgreSQL
```

External AI, search, email, URL, and media capabilities are called through service or utility boundaries rather than directly from route handlers.

## Responsibilities by layer

| Layer | Responsibility | Examples |
| --- | --- | --- |
| Router | Translate HTTP, validate input, enforce role dependencies | Auth, Verify, Reviewer, Admin |
| Schema | Define typed request, response, and model-output contracts | Pydantic models |
| Service | Coordinate business rules and multi-step workflows | Verification, review routing, account flows |
| Repository | Isolate database reads, writes, pagination, and aggregates | Claim, review, usage, audit repositories |
| Model | Map persistent entities and relationships | SQLAlchemy ORM models |
| Utility / adapter | Encapsulate external or cross-cutting mechanisms | URL fetcher, sanitisation, media probe, LLM compatibility |

This separation keeps route handlers small and prevents provider-specific AI behaviour from leaking into persistence code.

## Main bounded areas

### Verification

- Creates pending claim records.
- Normalises text, files, and URLs.
- Invokes media and AI workflows.
- Persists verdict, confidence, reasoning, media report, and AI usage.
- Invokes smart review routing after completion.

### Review

- Filters and prioritises pending queue items.
- Validates reviewer ground-truth submissions.
- Prevents duplicate reviews for a claim.
- Completes queue state within the review transaction.
- Aggregates calibration, confusion, agreement, and workload metrics.

### Authentication and administration

- Provide registration, activation, login, password recovery, role checks, moderation, invitations, and auditing.
- These are team-level system capabilities; they are included to explain architecture and are not claimed as my independent implementation.

## Asynchronous and concurrency boundaries

FastAPI and SQLAlchemy use asynchronous request and database interfaces. Network-bound verification work is parallelised where independence is safe:

- search across sub-claims;
- model assessments across configured providers; and
- SSE delivery while the pipeline continues.

Blocking SDK calls are moved away from the event loop. A thread-safe queue bridges the verification worker and asynchronous SSE generator.

Concurrency is bounded by pipeline stages rather than making every operation parallel. Claim aggregation, persistence, and reviewer routing remain ordered because they depend on completed prior state.

## Streaming consistency

The SSE path cannot simply delegate all persistence to the ordinary request path because it streams intermediate events while work is running. Its completion sequence is therefore explicit:

1. update claim verdict, confidence, and reasoning;
2. persist AI usage records;
3. update the query cache;
4. evaluate the smart review router;
5. commit the database transaction; and
6. send the `complete` event.

Committing before the final event avoids a race where the frontend receives a claim ID and immediately requests a record that is not yet visible.

## Data consistency

- UUIDs identify users, claims, reviews, and queue entries.
- A unique claim constraint enforces one review per claim in the prototype.
- Queue creation checks existing entries to remain idempotent.
- Foreign-key delete behaviour distinguishes retained history from dependent review data.
- Soft deletion supports claim moderation and restoration.
- Alembic versions every relational schema change and related data backfill.

## Reliability controls

- Typed Pydantic validation at HTTP and LLM-output boundaries.
- Global exception handling with traceable error identifiers.
- Explicit fallbacks when search, parsing, or one provider fails.
- Short-lived cache for repeated text-only verification.
- Structured access and application logs with rotation.
- Health endpoint with database-connectivity status.
- Media cleanup policies for temporary artefacts.

## Security boundaries

The final team system includes:

- bcrypt password hashing and strong password rules;
- JWT authentication with database revalidation;
- role-based access for reviewer and administrator routes;
- endpoint-specific rate limiting;
- prompt-input sanitisation;
- SSRF-aware URL fetching;
- CORS configuration and upload-size limits; and
- access and administrative audit logs.

Security is treated as a cross-cutting backend property. The existence of these team-level controls does not imply that every control was authored by the portfolio owner.

## Testing and delivery

The repository uses:

- pytest and pytest-cov for backend tests;
- in-memory SQLite fixtures for isolated service and route testing;
- frontend component tests with Vitest and Testing Library;
- Ruff and ESLint for automated checks;
- Docker Compose for PostgreSQL, backend, and frontend; and
- GitHub Actions for lint, test, build, and container checks.

## Transferable backend engineering concepts

VeriFact is implemented in Python/FastAPI, not Java. The engineering concepts map directly to common Java backend practices without presenting the project as a Java implementation:

| VeriFact implementation | Comparable Java/Spring concept |
| --- | --- |
| FastAPI router | Spring `@RestController` |
| Pydantic schema | Request/response DTO with Bean Validation |
| Service layer | Spring `@Service` |
| Repository layer | Spring Data repository / DAO |
| SQLAlchemy ORM | JPA / Hibernate |
| Alembic migration | Flyway / Liquibase |
| Dependency-based role guard | Spring Security filter / method security |
| Async SSE endpoint | Spring WebFlux or `SseEmitter` |
| Environment settings | Spring configuration properties |
| Docker Compose | Containerised multi-service development |

The portfolio therefore demonstrates API design, data modelling, transaction ordering, service layering, asynchronous processing, and deployment skills while remaining accurate about the implementation language.
