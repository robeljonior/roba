# ADR-003: Kafka in KRaft Mode

- **Status:** Accepted
- **Date:** 2026-11-16

## Context
ReliablePay needs an asynchronous event transport between the payment domain and webhook delivery. ZooKeeper is unnecessary for a modern Kafka deployment.

## Decision
Use Apache Kafka in KRaft mode for local development and the reference event pipeline. Publish only events represented by committed PostgreSQL outbox records. Use stable event IDs and payment IDs as keys when per-payment ordering is needed.

## Consequences
- Local Compose uses a single-node broker for convenience; this is not highly available.
- The outbox relay and consumers must tolerate duplicate delivery.
- Production deployments must define broker replication, ISR, retention, authentication, TLS, quotas, monitoring, and disaster recovery.
- Kafka transactions do not create a global transaction with PostgreSQL or merchant webhook endpoints.
