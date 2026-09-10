# SignalDesk - Implementation Roadmap

Version: 1.0

## Guiding rule

Do not start a later milestone because the current one became boring. Finish acceptance criteria first.

## M0 - Data feasibility spike

Target: 1-3 focused development days.

Deliverables:

- Rust CLI/command;
- Sectors/provider client skeleton;
- fetch 3-5 contrasting IDX stocks;
- verify units manually;
- calculate free-float market cap, float turnover, HHI, effective brokers, CR5, foreign-flow intensity;
- write JSON fixtures;
- document provider quota/cost behavior;
- decide exact historical free-float policy.

Exit criteria: all M0 criteria in SRS pass.

## M1 - Structural Liquidity MVP

Target: approximately 1-2 weeks of focused personal development.

### Backend

- Axum skeleton and config;
- PostgreSQL migrations;
- provider cache/source fetch table;
- instrument/market/free-float/broker/foreign-flow normalization;
- metric calculators + tests;
- radar build orchestration;
- methodology/classification versioning;
- REST endpoints;
- protected build operation.

### Frontend

- SvelteKit layout;
- radar table;
- filters/sorting;
- stock detail;
- broker distribution visualization;
- compare;
- methodology/provenance.

### Hardening

- recorded fixture integration test;
- offline read test;
- failed-build preservation;
- security check for provider key;
- basic structured tracing.

Exit criteria: M1 SRS acceptance criteria.

## M2 - Personal research workspace

Target: 1 week after M1 stabilizes.

- watchlist;
- personal notes;
- saved compare sets;
- structure metric history;
- market price/value history;
- simple daily dashboard.

Keep auth local/single-user unless public hosting requires more.

## M3 - Fundamentals and valuation

Target: 1-2 weeks depending provider coverage.

Backend:

- financial statement ingestion;
- normalized period model;
- TTM/annual handling;
- peer/sector mapping;
- valuation derivations.

Frontend:

- fundamentals tab;
- growth/profitability/leverage sections;
- valuation vs peer/historical range;
- no universal quality score initially.

## M4 - Market behavior and event intelligence

- relative return vs IHSG/sector;
- abnormal volume;
- volatility;
- corporate actions;
- filings/news timeline;
- event-coincident market behavior.

Do not state causality unless source/event relation is explicit.

## M5 - Alerts

- watchlist threshold rules;
- structural change alerts;
- event alerts;
- daily digest;
- background scheduler/worker if needed.

At this milestone, evaluate whether a separate worker process is justified.

## M6 - Grounded research assistant

- query SignalDesk stored data;
- explain formulas;
- compare companies;
- summarize recent changes;
- cite internal source facts in responses;
- refuse to invent unavailable data.

Use an LLM only after the deterministic data foundation is reliable.

## Engineering backlog, not milestones

Add only when evidence justifies:

- Redis;
- separate job queue;
- OpenTelemetry collector;
- multi-user auth;
- WebSocket/SSE build progress;
- TimescaleDB/partitioning;
- second market-data provider;
- mobile app.

## First ten implementation tasks

1. Initialize Rust backend and SvelteKit frontend repositories/workspace.
2. Add Docker Compose PostgreSQL.
3. Create config and secret loading.
4. Implement provider HTTP client and one verified endpoint.
5. Implement M0 CLI JSON output.
6. Write pure analytics functions with table/property tests.
7. Freeze normalized units and DB schema.
8. Implement source persistence/cache.
9. Implement radar build for one trading date.
10. Only then build the radar UI.
