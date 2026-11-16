# ADR-002: Durable Merchant-Scoped Idempotency

- **Status:** Accepted
- **Date:** 2026-11-16

## Context
HTTP clients retry after timeouts, gateways retry requests, and concurrent duplicate submissions are normal failure modes. An in-memory cache or a read-before-write check cannot prevent duplicate creation across instances.

## Decision
Require an `Idempotency-Key` for payment creation. Scope it by merchant and operation. Persist a canonical request fingerprint and the original response in PostgreSQL under a unique constraint on `(merchant_id, operation, idempotency_key)`.

- Same key + same request fingerprint: return the original semantic response with HTTP `200 OK`.
- Same key + different fingerprint: return `409 Conflict`.
- Concurrent same-key requests: exactly one request owns creation; others wait for or read the durable result.
- The first accepted request returns `201 Created`.
- Key retention is long-lived; do not expire a key while its payment may still be retried or reconciled.

## Consequences
- The database, not application memory, decides the winner.
- Request canonicalization must be deterministic and versioned.
- A crash during processing requires a durable in-progress state and recovery policy.
- The stored response must avoid credentials and sensitive information.
- Provider calls require their own stable provider idempotency key; merchant idempotency alone does not solve provider-side ambiguity.
