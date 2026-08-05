# Güngör Akbıyık

**Senior / Principal Java Engineer — 19+ years on high-throughput, real-time distributed systems.**

I build backends where the hard part is staying correct under load: high-concurrency
transaction engines, real-time data feeds, settlement and reconciliation pipelines, and
nationwide device fleets. The numbers below are ones I was accountable for, not estimates.

## What I've done

- Built a transactional platform from scratch as its first engineer — **500K transactions/day**, in production within 8 months (Java, Spring Boot, PostgreSQL, Kafka).
- Redesigned the sales path around a Redis cache and read-only replicas: **1,000 ms → 100 ms**.
- Moved a 5,000-dealer billing run onto Kafka event fan-out: **30 minutes → 2 minutes**.
- Stood in for missing distributed transactions across three services with sagas and compensating actions; idempotent queries and end-of-day reconciliation closed the double-spend gap.
- Collapsed duplicated security and configuration code into a **7-module Spring Boot starter family**; led the Boot 2.6 → 3.4 migration and the multi-tenant JWT standard.
- Designed and shipped central management and real-time monitoring for a **7,000+ device fleet**.
- Hired and led an **8-engineer backend team** from zero — then chose to go back to building.

## Selected work

**[core-banking](https://github.com/gungorakbiyik/core-banking)** — a core banking domain model
in Java 21 / Spring Boot 4, written to put DDD tactical patterns (value objects, entities,
aggregates, repositories, services) on a domain where correctness actually matters. Public from
day one, still in progress.

**[payment-service-demo](https://github.com/gungorakbiyik/payment-service-demo)** — a runnable
reproduction of an `@Async @EventListener` race condition: the listener reads the entity before
the transaction commits, then the commit overwrites what it wrote. Explains why the bug looks
intermittent, and fixes it with `@TransactionalEventListener`.

**[gungorakbiyik.github.io](https://github.com/gungorakbiyik/gungorakbiyik.github.io)** — my
site, built with Astro. Where the writing lives.

## Writing

Deep-dives on DDD, distributed systems, and the Java/Spring problems I actually run into.
**The articles are written in Turkish.**

- Domain-Driven Design — Level 1:
  [Entity & Value Object](https://gungorakbiyik.github.io/writings/ddd-level-1-part-1-entity-value-object) ·
  [Aggregate & Aggregate Root](https://gungorakbiyik.github.io/writings/ddd-level-1-part-2-aggregate) ·
  [Repository, Domain & Application Service](https://gungorakbiyik.github.io/writings/ddd-level-1-part-3-repository)
- [Account vs. Ledger — Foundations of Core Banking](https://gungorakbiyik.github.io/writings/account-vs-ledger-core-banking)

→ [gungorakbiyik.github.io](https://gungorakbiyik.github.io/) · also on [Medium](https://medium.com/@gungor.akbiyik)

## Stack

**Backend** Java 8–24 · Spring Boot · Spring Security · Spring Data JPA / Hibernate · REST · WebSocket
**Messaging** Apache Kafka · RabbitMQ
**Data** PostgreSQL · Oracle · Redis · Elasticsearch
**Platform** Docker · Kubernetes · Jenkins · Prometheus · Grafana · Keycloak
**Architecture** Microservices · Event-Driven (Saga) · Clean & Hexagonal · Domain-Driven Design

Currently learning Python and AI agents — I use AI tooling in my daily workflow rather than
reading about it.

## Contact

Istanbul, Turkey — open to remote, hybrid and on-site.

[Website](https://gungorakbiyik.github.io/) ·
[LinkedIn](https://www.linkedin.com/in/gungorakbiyik/) ·
[Medium](https://medium.com/@gungor.akbiyik) ·
gungor.akbiyik@gmail.com
