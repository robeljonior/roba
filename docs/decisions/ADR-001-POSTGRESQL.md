# ADR-001: PostgreSQL as the System of Record

- **Status:** Accepted
- **Date:** 2026-11-16

## Context
Payment correctness requires durable uniqueness constraints, transactions, row-level locking, referential integrity, and reliable recovery after process failure. A cache or message broker cannot replace the authoritative payment ledger.

## Decision
Use PostgreSQL as the source of truth for merchant, payment, idempotency, provider-operation, audit-history, and outbox data. Use database constraints and transactions to enforce invariants.

## Consequences
- Payment creation and state changes are transactionally durable.
- Unique constraints arbitrate concurrent idempotency claims.
- Backups, point-in-time recovery, migrations, connection pooling, and monitoring are operational requirements.
- Redis may be introduced for non-authoritative caching/rate limiting, but never as the sole idempotency authority.
- Kafka is an event transport, not the payment database.
