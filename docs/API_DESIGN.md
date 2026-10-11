# API design

[← Portfolio home](../README.md)

This document presents the public design surface without reproducing private course source code.

## Conventions

- Versioned base path: `/api/v1`
- JSON for standard requests and responses
- `multipart/form-data` for uploads
- Server-Sent Events for streamed verification progress
- UUID resource identifiers
- UTC, timezone-aware timestamps
- Bearer JWT for protected routes
- Consistent error body: `{"detail": "Human-readable message"}`

## Domain boundaries

| Domain | Responsibilities |
| --- | --- |
| Auth | Registration, activation, login, password reset, current user |
| Verify | Text and media verification, streaming, saved claims, reports |
| Reviewer | Review queue, decisions, history, AI-quality analytics |
| Admin | Users, moderation, invitations, audit and usage |

## Authentication model

Three request tiers are supported:

| Tier | Behaviour |
| --- | --- |
| Guest | Limited text verification; no private history |
| Authenticated user | Saved history and multimodal verification |
| Reviewer / administrator | Role-gated review or management routes |

Protected requests use `Authorization: Bearer <token>`. The backend does not trust the role claim alone: it checks the current database record so account bans and role changes take effect without waiting for an old token to expire.

## Verification contract

A text verification submission conceptually accepts:

```json
{
  "text": "A factual statement to verify"
}
```

The completed report contains:

```json
{
  "claim_id": "uuid",
  "status": "Supported | Refuted | Uncertain",
  "confidence_score": 0,
  "reasoning_chain": [
    {
      "concept": "A self-contained sub-claim",
      "verdict": "Supported",
      "confidence": 0,
      "evidence": "Evidence summary",
      "sources": ["https://source.example"],
      "reasoning": ["Reasoning step"],
      "disclaimer": ""
    }
  ]
}
```

Values above illustrate the shape; they are not a captured production response.

## Streaming contract

SSE is used for long-running text verification. The logical event types are:

| Event | Purpose |
| --- | --- |
| `stage_start` / `stage_done` | Announce pipeline-stage lifecycle |
| `partial` | Return a completed sub-claim while other work continues |
| `complete` | Return the final aggregated report |
| `error` | Return a safe error and tracing identifier |

Example:

```text
event: stage_done
data: {"type":"stage_done","stage":"extracting","message":"Sub-claims extracted"}

event: stage_start
data: {"type":"stage_start","stage":"searching","message":"Searching for evidence"}

event: partial
data: {"type":"partial","concept_idx":0,"data":{"verdict":"Refuted","confidence":76}}

event: complete
data: {"type":"complete","result":{"status":"Refuted","confidence_score":74}}
```

## Reviewer contract

Representative reviewer operations include:

```text
GET  /api/v1/reviewer/queue
POST /api/v1/reviewer/review
GET  /api/v1/reviewer/history
GET  /api/v1/reviewer/analytics
```

Queue reads support pagination and filters such as routing reason and confidence range. A review submission records a ground-truth verdict, optional notes, and an error category when the human verdict differs from the AI result.

## Error behaviour

| Status | Meaning |
| --- | --- |
| `400` | Invalid request state or token action |
| `401` | Missing or invalid authentication |
| `403` | Inactive account or insufficient role |
| `404` | Resource not found |
| `409` | Duplicate resource or conflicting state |
| `413` | Upload exceeds configured size |
| `422` | Schema validation failure |
| `429` | Rate limit exceeded |
| `500` | Unexpected failure with a traceable error identifier |

External search or model failures are handled inside the verification report where possible. A missing provider response should reduce evidence or consensus strength rather than cause the system to invent a successful result.

## Security controls

- Per-endpoint throttling for authentication and verification.
- Strong password validation and bcrypt hashing.
- Separate access, activation, and password-reset token purposes.
- Role-based dependencies for reviewer and administrator routes.
- Database revalidation of token subjects.
- Upload-size enforcement and media-type validation.
- SSRF-aware URL extraction.
- Prompt-input sanitisation.
- Structured access and audit logging.
