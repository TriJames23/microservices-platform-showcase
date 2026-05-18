# ADR-022: Idempotency Key Standard

## Status
**ACCEPTED** - 2026-03-17

## Context
Write-side operations can be retried by clients, gateways, or workers.
Without idempotency, duplicate commands create inconsistent state.

## Decision
### 1. When Required
Idempotency keys are **mandatory** for:
- Public write APIs (commands executed under JWT-based identity at public entrypoints)

Idempotency keys are **not required** for internal message consumption, because the Inbox pattern provides
deduplication (unique messageId + ignore duplicates) and retry semantics at the message boundary.

### 2. Key Format
- Header: `Idempotency-Key`
- Value: opaque string (UUID recommended)
- Minimum TTL: 24 hours

### 3. Storage and Scope
- Store key with request hash + result in a durable store
- Scope: per tenant + per endpoint + per principal
- Replays must return the original result

### 4. Conflict Handling
- If the same key is used with a different payload, return `409 Conflict`
- If key exists and payload matches, return cached response

### 5. Enforcement
Infrastructure pipeline enforces:
- Presence check (strict mode for public writes)
- Store/lookup
- Conflict detection (same key + different payload -> 409 Conflict)
- Replay (same key + same payload -> return original response)

## Where Idempotency Is Mandatory (Current Platform)

| Boundary | Mandatory? | Mechanism | Notes |
|---|---:|---|---|
| REST/gRPC public write commands | Yes (when strict mode enabled) | `IdempotencyBehavior` + store | Strictness is based on identity source = JWT. |
| REST/gRPC internal calls (X-* propagation) | No | N/A | Avoids forcing internal services to send Idempotency-Key. |
| Messaging consumer / Inbox receiver | No | Inbox dedupe | Uses unique messageId; duplicates ignored. |
| Saga steps | No | Inbox + saga state machine | Retried via message boundary; idempotency-store not required. |

## Consequences
- Safe retries
- Reduced duplicate processing

## References
- ADR-014: Saga State Machine Details
- ADR-016: DLQ and Error Handling
