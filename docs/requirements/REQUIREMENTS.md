# ReliablePay — Product and Engineering Requirements

**Status:** Phase 1 baseline  
**Scope:** Production-oriented reference implementation; simulated payment provider only  
**Version:** 1.0

## 1. Purpose

ReliablePay demonstrates how to build a payment orchestration service that remains correct when clients retry requests, requests race, processes restart, providers time out, and events are delivered more than once.

It is an educational and portfolio reference implementation. It is not a licensed payment service provider, does not connect to real banking rails, and must not process real funds or production customer data.

## 2. Goals

- Accept and manage payment-intent requests through a versioned HTTP API.
- Prevent duplicate payment creation for the same merchant idempotency key.
- Enforce legal payment state transitions and preserve an audit trail.
- Persist payment state and publishable events atomically using a transactional outbox.
- Deliver merchant webhooks at least once, with signatures, retries, and delivery history.
- Make failure behaviour observable and testable.
- Provide reproducible local infrastructure and automated CI.

## 3. Non-goals

- Real card acquiring, settlement, banking integrations, or custody of funds.
- PCI DSS certification or storage of PAN, CVV, PIN, or track data.
- Multi-region active-active writes in the initial release.
- Universal exactly-once delivery across PostgreSQL, Kafka, and merchant endpoints.
- A merchant dashboard, billing engine, FX conversion, or dispute-management system in v1.

## 4. Actors

| Actor | Responsibility |
|---|---|
| Merchant system | Creates payment requests and receives webhooks |
| ReliablePay API | Validates requests, enforces idempotency, owns payment state |
| Simulated provider | Produces deterministic success, decline, and timeout outcomes for tests |
| Outbox publisher | Relays committed outbox records to Kafka |
| Go webhook dispatcher | Delivers signed events and tracks retries |
| Operator | Inspects health, metrics, logs, and failed deliveries |

## 5. Functional requirements

### FR-01 — Create payment
The API shall accept an amount in integer minor units, an ISO-4217 currency, a merchant reference, and an idempotency key. It shall validate authentication/merchant ownership, amount bounds, currency support, and request shape before creating a payment.

### FR-02 — Idempotent request handling
The idempotency scope shall be merchant + operation + key. The service shall persist a canonical request fingerprint and the original response. A replay with the same fingerprint shall return the original semantic result with HTTP 200. Reuse of a key with a different fingerprint shall return HTTP 409. Concurrent requests shall result in at most one payment.

### FR-03 — Payment retrieval
An authorized merchant shall retrieve only its own payment by payment ID. Responses shall not expose internal secrets or provider credentials.

### FR-04 — Payment state machine
The domain shall reject illegal transitions. Every accepted transition shall be persisted with its actor, timestamp, prior state, new state, and reason.

### FR-05 — Provider ambiguity
A timeout or lost response from the simulated provider shall be represented as an unknown/pending outcome, not automatically treated as a decline. Reconciliation or a safe provider lookup shall resolve ambiguity.

### FR-06 — Durable event publication
A payment state change and its outbox event shall commit in one PostgreSQL transaction. A relay shall publish committed events to Kafka and mark them published only after broker acknowledgement. Relay crashes may cause duplicate publication.

### FR-07 — Webhook delivery
The dispatcher shall deliver events with a stable event ID, HMAC-SHA256 signature, timestamp, bounded timeout, retry policy, exponential backoff with jitter, and a terminal dead-letter state. A 2xx response acknowledges delivery. Non-2xx responses and network errors are retryable according to policy.

### FR-08 — Delivery visibility
The system shall retain delivery attempts, response status, next retry time, and final outcome without logging secrets or full sensitive payloads.

### FR-09 — Health and observability
Services shall expose liveness/readiness checks and structured logs with correlation IDs. Metrics shall include request counts/latency, idempotency replays/conflicts, outbox lag, webhook attempts, and dead-letter counts.

## 6. API baseline

Initial endpoints (exact schemas are finalized during implementation):

| Method | Path | Purpose |
|---|---|---|
| POST | /api/v1/payments | Create or replay a payment |
| GET | /api/v1/payments/{paymentId} | Retrieve a payment |
| GET | /actuator/health/liveness | Liveness |
| GET | /actuator/health/readiness | Readiness |

Create requests require an `Idempotency-Key` header. API errors use a consistent machine-readable problem format. Authentication is merchant-scoped; local development may use seeded test credentials, never hard-coded production secrets.

## 7. Money and currency

- Store amounts as signed 64-bit integer minor units; reject zero/negative amounts unless a future operation explicitly supports them.
- Never use binary floating point for monetary values.
- Support ETB (2 decimal places) and USD (2 decimal places) initially.
- Currency is immutable after payment creation.
- No FX conversion is performed.

## 8. Non-functional requirements

| Area | Requirement |
|---|---|
| Correctness | At most one payment per merchant/operation/idempotency key |
| Durability | A successful API response is backed by committed PostgreSQL data |
| Consistency | Payment, idempotency result, audit entry, and outbox event commit atomically where applicable |
| Security | TLS at deployment edge; merchant-scoped authorization; secrets via environment/secret manager |
| Resilience | Bounded timeouts, retries only where safe, graceful shutdown |
| Performance | Establish and publish a local baseline; do not claim an unmeasured SLA |
| Maintainability | Domain logic isolated from HTTP, persistence, and provider adapters |
| Testability | Unit, repository, API, concurrency, and failure-injection tests |
| Portability | Docker Compose local environment and documented configuration |

## 9. Acceptance criteria

1. Repeating an identical create request with the same key returns the same payment ID and does not create a second payment.
2. Reusing a key with a changed amount or currency returns 409 and leaves the original payment unchanged.
3. A concurrency test with many identical requests creates exactly one payment.
4. A simulated provider timeout does not produce a false final decline.
5. Payment state, audit entry, and outbox record cannot be partially committed.
6. Restarting the outbox relay does not lose committed events; duplicates are safely tolerated.
7. A webhook receiver can verify the signature and deduplicate by event ID.
8. Failed webhook attempts are visible and eventually dead-lettered according to policy.
9. CI runs formatting, compilation, unit tests, and integration tests.
10. Documentation clearly states the project is not a real payment processor.

## 10. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Duplicate charge due to retries | Durable unique idempotency constraint; provider-side idempotency in a future real adapter |
| Lost event after database commit | Transactional outbox |
| Duplicate event delivery | Stable event ID and consumer deduplication |
| Provider timeout ambiguity | Explicit pending/unknown state and reconciliation |
| Cross-merchant data exposure | Merchant-scoped authorization and negative tests |
| Secret leakage | Secret scanning, redacted logs, environment-based configuration |
| False confidence from local tests | Document limits; add failure-injection and concurrency tests |

## 11. Delivery phases

- **Phase 1:** Requirements, architecture, data model, and architecture decision records.
- **Phase 2:** Java API, domain model, PostgreSQL migrations, idempotency, and integration tests.
- **Phase 3:** Transactional outbox relay and Kafka event flow.
- **Phase 4:** Go webhook dispatcher, signing, retries, and dead-letter handling.
- **Phase 5:** Security, observability, resilience tests, CI, and operational documentation.
