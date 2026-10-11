# My contributions — Kunyu Li (`like-rich`)

[← Portfolio home](../README.md) · [中文首页](../README.zh-CN.md)

## Scope

VeriFact was a team capstone project. This page separates my individual engineering work from the capabilities of the final team product.

The record below was prepared by reviewing the private repository's authored commits, feature branches, merge records, and file-level diffs associated with `like-rich` / Kunyu Li. Personal email addresses and private repository links are intentionally omitted.

## Contribution map

| Area | My role | Main layers affected |
| --- | --- | --- |
| AI fact-checking agent | Primary implementation of the core verification workflow | FastAPI services, schemas, configuration |
| Evidence retrieval and quality | Multi-query search, source classification, evidence annotation | Agent and trusted-source utilities |
| Multi-model reasoning | Provider abstraction, structured assessments, weighted consensus | Agent orchestration |
| Reviewer workflow | End-to-end design and implementation | Database, migrations, repository, service, API, React |
| Reviewer analytics | Metrics design, aggregation and visualisation | SQL queries, service layer, dashboard |
| Integration | Connect review routing to verification and navigation | Backend and frontend integration |
| Project documentation | Architecture, setup and deployment documentation | Root documentation |

## 1. AI verification pipeline

I developed the core pipeline that turns an unstructured user statement into an explainable result.

### Claim decomposition

- Parse the input into 1–5 self-contained factual sub-claims.
- Keep the sub-claims specific enough to retrieve evidence.
- Preserve the language of the user's input in the generated explanation.

### Evidence retrieval

- Generate multiple search queries for each sub-claim.
- Execute retrieval work in parallel to reduce end-to-end latency.
- Support web-search evidence and direct URL content.
- Retry with broader queries when initial evidence is weak.
- Treat empty or unusable evidence as uncertainty, not as permission to guess.

### Structured assessment

Each model assessment returns a validated structure rather than free-form text:

```text
verdict · confidence · explanation · reasoning steps
evidence summary · source URLs · search quality · disclaimer
```

This contract makes model outputs persistable, comparable, renderable, and suitable for later human review.

## 2. Evidence quality and consensus

I implemented source-aware evidence processing and cross-model consensus.

### Source-aware verification

- Classify and annotate source credibility before the evidence reaches the judging prompt.
- Prefer first-party, institutional, academic, and established editorial sources where available.
- Remove excluded low-quality links from the final source list.
- Instruct the model to return no URL rather than fabricate one.

### Capability-weighted consensus

For each sub-claim, configured providers evaluate the same evidence independently. Their results are grouped into `Supported`, `Refuted`, and `Uncertain`, then combined using provider capability weights and confidence.

The consensus logic includes:

- a weighted two-thirds supermajority path;
- a lower-confidence majority path when no supermajority exists;
- penalties for disagreement;
- explicit uncertainty when evidence or confidence is too weak; and
- aggregation of per-claim consensus into a final claim verdict.

This design keeps disagreement observable instead of hiding it behind a single model response.

## 3. Reviewer workflow

I designed and implemented the reviewer feature across the complete application stack.

### Backend and database

- Created review and review-queue entities with versioned Alembic migrations.
- Added repository queries for queue filtering, history, analytics, and status updates.
- Implemented service-layer rules for routing, ground-truth submission, agreement detection, and duplicate-review protection.
- Exposed reviewer APIs with role checks, pagination, validation, and consistent response schemas.
- Connected queue routing to both synchronous and SSE verification completion paths.

### Smart queue routing

Results enter the queue when they match at least one quality-control rule:

| Rule | Purpose |
| --- | --- |
| Confidence below 70% | Prioritise predictions the model itself considers weak |
| `Uncertain` verdict | Ensure unresolved claims receive human attention |
| Conflicting sub-claim evidence | Surface internally inconsistent results |
| 15% sample of high-confidence results | Detect confident errors and monitor calibration |

Queue priority increases as confidence decreases. A unique claim constraint prevents the same result from being reviewed multiple times.

### Reviewer experience

- Built the queue, decision, history, and analytics interfaces.
- Presented the original claim, AI result, confidence, sources, and reasoning together.
- Supported ground-truth labels, reviewer notes, and disagreement categories.
- Integrated reviewer navigation without exposing a user's private verification history.

## 4. Evaluation dashboard

I built the data aggregation and visualisation needed to inspect system behaviour rather than relying on an overall accuracy number.

| Metric | Question answered |
| --- | --- |
| AI–human agreement | How often does the reviewer accept the AI verdict? |
| False-confidence rate | How often is a high-confidence result rejected? |
| Confusion matrix | Which verdict classes are confused with each other? |
| Calibration curve | Does stated confidence correspond to observed agreement? |
| Error distribution | Are failures caused by evidence, reasoning, or source bias? |
| Queue metrics | How large and how old is the outstanding review workload? |

## 5. Integration and documentation

- Refined reviewer UI state, report integration, and responsive styling after the initial feature delivery.
- Updated the application routes, sidebar, verification API integration, composer, report, and workspace behaviour needed for a coherent reviewer flow.
- Authored the project-level README covering technology choices, architecture, configuration, local setup, and Docker deployment.

## Private history reviewed

The following authored milestones were found in the private course repository:

| Date | Authored milestone |
| --- | --- |
| 27 Jun 2026 | Integrated search tooling and structured parsing into the backend agent |
| 11 Jul 2026 | Completed the fact-checking backend and core AI-agent logic |
| 22 Jul 2026 | Delivered the reviewer role, workflow, and evaluation dashboard |
| 22 Jul 2026 | Refined and integrated the reviewer module |
| 24 Jul 2026 | Added root architecture and local-setup documentation |

These milestones are supported by three pull-request merge records made through the `like-rich` GitHub account. The source and commit pages remain private.

## Ownership boundary

The finished product also contains substantial work by other team members. I do **not** claim sole authorship of the complete frontend, authentication and administration foundations, media-processing pipeline, CI/CD hardening, or every final integration change. Those capabilities appear in the main portfolio because they explain the final system, while the sections above identify my own work specifically.

This distinction is intentional: the project demonstrates both my individual engineering contribution and my ability to integrate it into a larger team-owned system.
