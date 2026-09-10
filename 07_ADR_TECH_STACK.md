# ADR-001 - Technology Stack for SignalDesk

Status: Accepted  
Decision date: 10 September 2026

## Context

SignalDesk is now a personal long-term project rather than a deadline-driven hackathon submission. The system needs reliable market-data ingestion, deterministic analytics, a research-oriented web UI, historical PostgreSQL storage, and room to grow into fundamentals/events/alerts later.

The developer is professionally strongest in Go but wants a real project for learning Rust. Delivery speed still matters, but there is no three-week submission constraint.

## Decision

Use:

- **Rust + Axum** for backend HTTP/application code;
- **Tokio** for async runtime;
- **Reqwest** for provider calls;
- **SQLx + PostgreSQL** for persistence;
- **Serde** for data contracts;
- **Tracing** for observability;
- **SvelteKit + TypeScript** for frontend;
- **Tailwind + shadcn-svelte** for UI primitives;
- **Apache ECharts** for charts;
- **Docker** for reproducible local/deployed setup.

Use a **modular monolith**, not microservices.

## Why Rust

Advantages:

- strong compile-time type guarantees around domain data;
- explicit error handling;
- mature async/network ecosystem;
- predictable runtime footprint;
- valuable learning payoff for a long-lived personal project.

Costs:

- slower initial development than Go for this developer;
- longer compile cycles;
- ownership/lifetime friction while learning;
- more ecosystem decisions.

The costs are accepted because the project has no hard hackathon deadline and learning Rust is itself part of the value.

## Why Axum

Axum aligns naturally with Tokio/Tower, has straightforward routing/extractors/middleware, and is sufficient for a REST research application without adding framework-specific runtime concepts.

## Why PostgreSQL instead of SQLite

The product now targets long-term historical ingestion, snapshots, joins across multiple data domains, future watchlists/notes, and analytics queries. PostgreSQL avoids an early storage migration and remains operationally simple enough for a personal project.

## Why SvelteKit

SvelteKit provides a low-ceremony TypeScript UI stack suitable for data-heavy research pages. It reduces the amount of component/state boilerplate compared with larger React setups while retaining routing, SSR/static options, and strong ecosystem support.

## Explicitly deferred

- Redis;
- Kafka/NATS/RabbitMQ;
- Kubernetes;
- microservices;
- GraphQL;
- event sourcing;
- TimescaleDB;
- separate worker service.

These are not banned forever. They require a measured problem first.

## Consequences

Positive:

- one coherent backend language/runtime;
- strong domain-modeling opportunity;
- durable database from day one;
- clean path to background worker split later;
- frontend remains independently deployable.

Negative:

- M0/M1 will take longer than an equivalent Go implementation;
- Rust learning can become scope creep if the developer rewrites abstractions for elegance rather than shipping features.

## Guardrail

When choosing between clever Rust abstraction and a boring explicit implementation, choose the boring explicit implementation until M1 works.
