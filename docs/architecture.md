# Architecture and design decisions

[← 中文首页](../README.md) · [English home](../README_EN.md)

## Context

VeriFact accepts heterogeneous inputs, performs network and model calls with variable latency, and must present uncertain results honestly. The architecture separates deterministic application behaviour from probabilistic AI behaviour and adds human review as a first-class subsystem.

## Component view

```mermaid
flowchart LR
    subgraph Actors
        USER[Guest / user]
        REVIEWER[Reviewer]
        ADMIN[Administrator]
    end

    subgraph Frontend[React SPA]
        WORKSPACE[Verification workspace]
        REPORT[Explainable report]
        REVIEW_UI[Queue, history, analytics]
        ADMIN_UI[Administration]
    end

    subgraph API[FastAPI application]
        MIDDLEWARE[CORS, rate limits, logging, errors]
        ROUTERS[Auth, Verify, Reviewer, Admin]
        SERVICES[Business services]
        AGENT[AI agent pipeline]
        MEDIA[Media pipeline]
        REPOSITORIES[Repository layer]
    end

    subgraph Integrations
        SEARCH[Search providers]
        LLMS[LLM providers]
        MEDIA_TOOLS[Local media tools]
        EMAIL[Email provider]
        DB[(PostgreSQL)]
        FILES[(Temporary media storage)]
    end

    USER --> WORKSPACE
    WORKSPACE --> REPORT
    REVIEWER --> REVIEW_UI
    ADMIN --> ADMIN_UI
    WORKSPACE --> MIDDLEWARE
    REVIEW_UI --> MIDDLEWARE
    ADMIN_UI --> MIDDLEWARE
    MIDDLEWARE --> ROUTERS --> SERVICES
    SERVICES --> AGENT
    SERVICES --> MEDIA
    SERVICES --> REPOSITORIES
    AGENT --> SEARCH
    AGENT --> LLMS
    MEDIA --> MEDIA_TOOLS
    MEDIA --> FILES
    SERVICES --> EMAIL
    REPOSITORIES --> DB
```

## Verification sequence

```mermaid
sequenceDiagram
    actor User
    participant UI as React client
    participant API as FastAPI
    participant Media as Media pipeline
    participant Agent as Agent orchestrator
    participant Search as Evidence search
    participant LLM as LLM providers
    participant DB as PostgreSQL
    participant Review as Smart review router

    User->>UI: Submit text, file or URL
    UI->>API: REST/multipart request
    API->>DB: Create pending claim
    opt Media input
        API->>Media: Extract transcript, frames and provenance
        Media-->>API: Normalised text and media report
    end
    API->>Agent: Verify normalised claim
    Agent->>Agent: Decompose into sub-claims
    par Each sub-claim
        Agent->>Search: Run multiple queries
        Search-->>Agent: Evidence and source URLs
        Agent->>LLM: Structured parallel assessments
        LLM-->>Agent: Verdicts, confidence and reasoning
    end
    Agent->>Agent: Weighted consensus and aggregation
    Agent-->>API: Explainable report
    API->>DB: Persist result and usage
    API->>Review: Apply review-routing rules
    Review->>DB: Optionally create queue item
    API-->>UI: SSE completion / response
```

## Key decisions

### Evidence before judgement

The judging model receives retrieved evidence rather than being asked to answer from parametric memory alone. Evidence is annotated with source-quality context, and final URLs must originate from the retrieved material.

**Trade-off:** retrieval adds latency and inherits search-engine coverage gaps. The system exposes uncertainty instead of hiding those gaps.

### Decompose before verifying

A single input can mix several factual assertions. Decomposition provides a verdict and evidence trail for each assertion before producing an overall result.

**Trade-off:** poor decomposition can distort later stages, so the structured output is bounded to a small number of self-contained sub-claims.

### Parallel model assessments

Configured providers assess the same sub-claim and evidence concurrently. The system retains model-level results, applies capability weights, and lowers confidence when models disagree.

**Trade-off:** multiple providers increase cost and operational variability. Usage metadata is recorded so cost and reliability can be monitored.

### Server-Sent Events for progress

Verification is server-driven and mainly one-way after submission, making SSE simpler than a bidirectional WebSocket channel. Stage and partial-result messages keep the UI responsive during network-bound work.

**Trade-off:** reconnection, cancellation, and intermediary timeouts require explicit handling in a production deployment.

### Human review as quality control

The review queue accepts risky cases and a random sample of apparently safe ones. Sampling high-confidence results is important because reviewing only uncertain outputs cannot measure overconfidence.

**Trade-off:** the 70% threshold and 15% sample rate are prototype policy values and should be tuned using workload and calibration data.

### Layered backend

Routers translate HTTP, services own business rules, repositories isolate data access, and ORM models define persistence. AI and external-service adapters remain behind service boundaries.

**Trade-off:** this introduces more files and interfaces than a small prototype strictly needs, but it supports testing and independent feature evolution.

## Reliability and security boundaries

- Pydantic schemas validate API and model-output structures.
- Model parsing or provider failures become explicit uncertain branches.
- JWTs are revalidated against the database so bans and role changes take effect immediately.
- Rate limits protect registration, login, and verification endpoints.
- URL fetching applies SSRF-aware validation.
- Prompt input is sanitised before interpolation.
- Media uploads are size-limited and temporary artefacts are cleaned up.
- Logs and audit records are separated from user-facing error details.
- Secrets are injected through environment variables and are not stored in this portfolio.

## Deployment topology

The capstone environment uses Docker Compose with three primary services:

```mermaid
flowchart LR
    B[Browser] --> FE[Frontend container :5173]
    FE --> BE[Backend container :8223]
    BE --> DB[(PostgreSQL 16)]
    BE -. HTTPS .-> EXT[Search, LLM and email APIs]
    BE -. process calls .-> TOOLS[Optional local media tools]
```

The backend runs schema migrations before application startup. A production evolution would place a reverse proxy in front, use managed PostgreSQL and object storage, move heavy media work to background workers, and add traces and queue-level observability.
