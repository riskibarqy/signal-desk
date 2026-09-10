# SignalDesk Documentation Pack

Version: 1.0  
Technology baseline: Rust + Axum + PostgreSQL + SvelteKit  
Product mode: Personal, long-term research application  
Primary market: Indonesia-listed equities (IDX)

## Documents

1. **01_PRD.md** - Product Requirements Document. Defines the problem, users, product scope, product principles, MVP, roadmap, success criteria, and non-goals.
2. **02_SRS.md** - Software Requirements Specification. Defines implementable functional and non-functional requirements, calculations, data quality, APIs, persistence, security, testing, and acceptance criteria.
3. **03_ARCHITECTURE.md** - System architecture and engineering decisions for a modular monolith using Rust/Axum, PostgreSQL, background ingestion, and SvelteKit.
4. **04_DATA_ANALYTICS_SPEC.md** - Data model, units, derivations, structural-liquidity formulas, classifications, quality flags, and future analytics extensions.
5. **05_API_SPEC.md** - REST/JSON contract, conventions, endpoints, examples, error model, pagination, caching metadata, and admin operations.
6. **06_ROADMAP.md** - Development milestones from data feasibility to structural-liquidity MVP, research pages, fundamentals, events, alerts, and later AI-assisted research.
7. **07_ADR_TECH_STACK.md** - Architecture Decision Record explaining why Rust/Axum + SvelteKit + PostgreSQL was chosen and what is deliberately not included initially.

## Recommended implementation order

1. Read the PRD once.
2. Treat the SRS as the implementation contract for milestone M1.
3. Build the data-feasibility spike before any polished UI.
4. Follow the architecture document unless real data invalidates an assumption.
5. Implement formulas from the Data & Analytics Specification as pure functions with tests.
6. Keep the API aligned with the API Specification; generate OpenAPI from Rust code once routes stabilize.
7. Use the roadmap to prevent random feature expansion.

## Product boundary

SignalDesk is a research and market-structure analysis tool. It does not provide investment advice, execute trades, accuse market participants of manipulation, infer beneficial owners from broker codes, or claim to reproduce proprietary IDX/MSCI methodologies.
