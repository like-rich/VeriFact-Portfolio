# API design: implementation record and design review

[← 中文首页](../README.md) · [English home](../README_EN.md)

This document separates behaviour verified against the private course implementation from recommended production improvements. It reproduces interface shapes, not source code.

## Part I — verified implementation

### Conventions

- Base path: `/api/v1`
- JSON for ordinary structured bodies and responses
- `multipart/form-data` for login and verification submissions
- Server-Sent Events (SSE) for streamed text verification
- UUID resource identifiers
- Bearer JWT for protected routes
- Standard FastAPI error shape: `{"detail": "Human-readable message"}`

### API domains

| Prefix | Main responsibilities |
| --- | --- |
| `/api/v1/auth` | Registration, activation, login, password reset, reviewer invitation acceptance |
| `/api/v1/verify` | Text and multimodal verification, streaming, history, detail, soft deletion |
| `/api/v1/reviewer` | Queue, ground-truth submission, history, AI-quality analytics |
| `/api/v1/admin` | Statistics, users, moderation, invitations, audit records |

## Text verification

### `POST /api/v1/verify/text`

Guest-accessible text verification with a 10 requests/minute IP rate limit.

```http
Content-Type: multipart/form-data
```

| Form field | Type | Required | Constraint |
| --- | --- | --- | --- |
| `text_content` | string | Yes | 1–10,000 characters |

Equivalent command-line request:

```bash
curl -X POST http://localhost:8223/api/v1/verify/text \
  -F 'text_content=A factual statement to verify'
```

The endpoint sanitises the text, creates a claim with a nullable user ID for guests, runs verification, persists the result, and returns a structured report.

### `POST /api/v1/verify/text/stream`

Uses the same `multipart/form-data` field and 10 requests/minute limit, but responds with `text/event-stream`.

Verified event names:

| Event | Purpose |
| --- | --- |
| `loaded` | A cached result was found |
| `stage_start` | A pipeline stage has started |
| `stage_done` | A pipeline stage has completed |
| `partial` | One sub-claim result is available |
| `complete` | The final persisted result is available |

Example frames:

```text
event: stage_start
data: {"type":"stage_start","stage":"searching","message":"Searching for evidence","detail":{}}

event: partial
data: {"type":"partial","stage":"partial","concept_idx":0,"data":{"verdict":"Refuted","confidence":76}}

event: complete
data: {"type":"complete","stage":"complete","result":{"status":"Refuted","confidence_score":74}}
```

Before emitting `complete`, the implementation writes the claim result and usage data, updates the short-lived cache, invokes review routing, and commits the database transaction.

## Multimodal verification

### `POST /api/v1/verify/claim`

Requires an active authenticated user and is limited to 20 requests/minute.

```http
Authorization: Bearer <access-token>
Content-Type: multipart/form-data
```

| Form field | Type | Required | Purpose |
| --- | --- | --- | --- |
| `text_content` | string | Conditional | Text, maximum 10,000 characters |
| `media_file` | binary file | Conditional | Image, audio, or video upload |
| `url_link` | string | Conditional | External page or supported video URL |

At least one of the three fields must be supplied.

## Verification response

The verified response schema includes:

```json
{
  "claim_id": "uuid",
  "text_content": "A factual statement to verify",
  "url_link": null,
  "media_path": null,
  "media_type": "text",
  "media_report": null,
  "status": "Supported",
  "confidence_score": 84,
  "reasoning_chain": [
    {
      "concept": "A self-contained sub-claim",
      "evidence": "Evidence summary",
      "reasoning": ["Reasoning step"],
      "explanation": "Plain-language explanation",
      "disclaimer": null,
      "conclusion": "Sub-claim conclusion",
      "search_quality": "relevant",
      "verdict": "Supported",
      "confidence": 84,
      "sources": ["https://source.example"],
      "model_results": [],
      "consensus_verdict": "Supported",
      "consensus_confidence": 84
    }
  ],
  "models_used": ["provider-model-key"],
  "created_at": "2026-07-30T12:00:00Z"
}
```

Values are illustrative; field names and types are based on the implemented Pydantic schema.

## Saved-claim operations

```text
GET    /api/v1/verify/history?skip=0&limit=10
GET    /api/v1/verify/{claim_id}
DELETE /api/v1/verify/{claim_id}
```

All require an active user. Detail and deletion apply an ownership check; deletion is soft deletion.

## Reviewer operations

```text
GET  /api/v1/reviewer/queue
POST /api/v1/reviewer/review
GET  /api/v1/reviewer/history
GET  /api/v1/reviewer/analytics
```

Reviewer and administrator roles are accepted. Queue filters include routing reason and confidence bounds.

The review request is JSON:

```json
{
  "claim_id": "uuid",
  "ground_truth_verdict": "supported",
  "error_type": null,
  "reviewer_notes": "Optional notes"
}
```

`ground_truth_verdict` accepts `supported`, `refuted`, `uncertain`, or `insufficient_evidence`. When supplied, `error_type` accepts `evidence_hallucination`, `reasoning_error`, or `source_bias`.

## Implemented authentication behaviour

Access-token authentication currently works as follows:

1. Login accepts OAuth2-style form fields `username` and `password`.
2. The issued JWT contains `sub`, `role`, and `exp`.
3. Protected routes decode the JWT with the shared JWT secret and algorithm.
4. Active-user dependencies parse `sub` as a UUID and reload the user from PostgreSQL.
5. The current database role replaces the token role, so bans and role changes take effect immediately.

Activation, password-reset, and reviewer-invitation JWTs add a `purpose` claim and their dedicated decode functions check that value.

## Implemented rate limits

| Endpoint | Limit |
| --- | --- |
| `POST /auth/register` | 3/hour |
| `POST /auth/login` | 5/minute |
| `POST /verify/text` | 10/minute |
| `POST /verify/text/stream` | 10/minute |
| `POST /verify/claim` | 20/minute |

## Error behaviour

| Status | Implemented meaning |
| --- | --- |
| `400` | Invalid state, empty sanitised claim, or invalid action token |
| `401` | Missing, invalid, expired, deleted-user, or inactive-user authentication |
| `403` | Insufficient role or access to another user's claim |
| `404` | Claim or review target not found |
| `409` | Registration conflict |
| `413` | Upload exceeds configured request limit |
| `422` | Form, UUID, or schema validation error |
| `429` | Rate limit exceeded |
| `500` | Unexpected server failure; global handling may attach an error ID |

## Part II — design review and recommended improvements

### Token-purpose separation

The implementation uses one signing secret for access, activation, password-reset, and invitation JWTs. Special-purpose decoders validate `purpose`, but the Bearer access-token dependency does **not** require `purpose=access`; access tokens themselves do not currently contain that claim.

This creates a token-type confusion boundary: a valid signed activation or password-reset token with a UUID subject can reach the access-token path. Database revalidation blocks inactive or missing users, but an active user's non-access token should still never be accepted as a session credential.

Recommended hardening:

1. Add `purpose: "access"`, `iss`, and `aud` to access tokens.
2. Require all three claims in the Bearer dependency.
3. Use separate signing keys or derived keys for access and action tokens.
4. Add unique `jti` values and one-time consumption for reset and activation tokens.
5. Add regression tests proving each token type is rejected by every other verification path.

### Contract cleanup

- Publish generated OpenAPI as a versioned artefact and validate examples against it.
- Remove duplicate field declarations in legacy-compatible schemas.
- Use a stable error envelope with `code`, `detail`, and `error_id` across validation and application failures.
- Document maximum upload types as well as maximum size.

### Streaming reliability

- Propagate client disconnect cancellation to the worker.
- Replace request-local threads with a durable background job queue for production.
- Persist stage state for reconnect and replay.
- Avoid best-effort exception swallowing around final persistence and queue routing.

### Reviewer consistency

- Claim queue items transactionally or lease them to prevent simultaneous review.
- Return `409 Conflict` for a second review rather than mapping all service `ValueError` cases to `404`.
- Add reviewer-scoped history if global history is not intended for every reviewer.
