# SignalDesk - System Architecture

Version: 1.0  
Architecture style: Modular monolith  
Backend: Rust/Axum  
Frontend: SvelteKit  
Database: PostgreSQL

## 1. Architecture goals

- simple enough for one developer;
- correct and reproducible market-data calculations;
- external-provider outages do not break read pages;
- provider-specific data stays behind adapters;
- background ingestion and user reads are separated logically;
- historical snapshots can be reproduced by methodology version;
- avoid infrastructure that does not solve a measured problem.

## 2. Topology

```text
                      +------------------+
                      |   SvelteKit Web   |
                      +---------+--------+
                                |
                           REST/JSON
                                |
                      +---------v--------+
                      |    Rust / Axum   |
                      |  Modular Monolith|
                      +---------+--------+
                                |
         +----------------------+----------------------+
         |                      |                      |
 +-------v-------+      +-------v-------+      +-------v-------+
 | Read Services |      | Build/Ingest  |      | Methodology   |
 +-------+-------+      +-------+-------+      +---------------+
         |                      |
         +-----------+----------+
                     |
              +------v-------+
              | PostgreSQL   |
              +------+-------+
                     ^
                     |
              +------+-------+
              | Provider     |
              | Adapters     |
              +------+-------+
                     |
                  Sectors
```

## 3. Modular boundaries

Start as one Rust crate/binary unless compile-time or ownership pressure justifies splitting.

Suggested source layout:

```text
backend/
  src/
    main.rs
    config.rs
    api/
      mod.rs
      routes/
      middleware/
      dto/
    domain/
      instrument.rs
      market.rs
      structure.rs
      quality.rs
      methodology.rs
    application/
      radar.rs
      stock_detail.rs
      compare.rs
      build.rs
    analytics/
      float.rs
      broker.rs
      foreign_flow.rs
      classification.rs
    provider/
      mod.rs
      sectors/
    repository/
      mod.rs
      postgres/
    jobs/
      build_radar.rs
    observability.rs
    error.rs
  migrations/
```

Split into Cargo workspace crates only if one of these becomes true:

- provider client is reused by another application;
- analytics becomes independently versioned;
- compile times become painful;
- separate worker binary needs a truly independent dependency graph.

## 4. Request path

Normal user reads:

```text
Browser -> SvelteKit -> Axum -> Application Service -> PostgreSQL
```

No provider call occurs on this path.

## 5. Ingestion/build path

```text
Operator/Scheduler
      |
      v
Build Service
      |
      +--> Provider cache lookup
      |
      +--> Provider call if needed
      |
      +--> Persist raw/source metadata
      |
      +--> Normalize
      |
      +--> Validate units/dates
      |
      +--> Calculate metrics
      |
      +--> Cross-sectional classification
      |
      +--> Commit completed radar snapshot
```

## 6. Provider abstraction

Sectors is the first provider but not the domain model.

Rules:

- provider DTOs live under `provider/sectors`;
- application/domain layers consume provider-independent input structs;
- units are converted at adapter/normalization boundary;
- source metadata is retained for provenance;
- retry policy stays in provider infrastructure, not analytics.

## 7. Background jobs

M1 can run builds in the API process through a controlled task because usage is personal and low-volume.

Do not immediately add Redis/queue infrastructure.

When jobs become long-running or scheduled reliably, split a `worker` binary that uses the same application/repository modules. Use PostgreSQL advisory locks or a job lease table before introducing another system.

## 8. PostgreSQL design

PostgreSQL is both:

- normalized system of record;
- persistent provider cache/history.

Use indexes around:

- `(instrument_id, trading_date)`;
- `trading_date` for radar/date queries;
- `cache_key` unique lookup;
- `(trading_date, metric_version)`;
- `(radar_id, instrument_id)`.

Avoid premature partitioning. Daily data for IDX is small by PostgreSQL standards.

## 9. Transaction model

Radar snapshot semantics:

1. insert `building` snapshot;
2. ingest/calculate data;
3. write members and metrics;
4. atomically transition to `completed`;
5. readers select completed snapshots only.

A failed build remains `failed`. Never overwrite a previous completed snapshot in-place.

## 10. Concurrency

Rust/Tokio rules:

- use bounded `Semaphore` around provider calls;
- prefer `FuturesUnordered`/buffered stream with fixed concurrency rather than spawning unlimited tasks;
- avoid holding DB transactions across external HTTP requests;
- use separate short transactions for source persistence and final radar commit;
- propagate tracing span/build ID into tasks.

## 11. Error model

Internal error categories:

- `ValidationError`;
- `NotFound`;
- `Conflict`;
- `ProviderError { retryable }`;
- `RepositoryError`;
- `DataQualityError`;
- `InternalError`.

Use typed errors internally. Convert to stable API error codes at the HTTP boundary.

## 12. Configuration

Environment/config fields:

- database URL;
- provider base URL;
- provider API key;
- provider timeout;
- provider concurrency;
- provider retry count;
- allowed CORS origins;
- admin token/secret for local first version;
- log level;
- build feature flags;
- methodology defaults.

Secrets never enter checked-in config files.

## 13. Observability

Use `tracing` spans:

- HTTP request span;
- radar build span;
- provider request span;
- DB operation span for slow queries;
- analytics summary span.

Logs should be structured JSON in deployed mode and pretty text locally.

Potential future OpenTelemetry export can be added without changing business code.

## 14. Frontend architecture

Suggested SvelteKit layout:

```text
web/src/
  routes/
    +page.svelte
    radar/+page.svelte
    stock/[symbol]/+page.svelte
    compare/+page.svelte
    methodology/+page.svelte
  lib/
    api/
    components/
    charts/
    format/
    types/
    stores/
```

Prefer server load functions for initial page data when convenient, then client-side sorting/filtering for already-loaded tables.

Do not duplicate analytics formulas in TypeScript. The backend is the source of truth.

## 15. Deployment

Local development:

```text
Docker Compose: PostgreSQL
cargo run: API
pnpm dev: SvelteKit
```

Production first version:

- one PostgreSQL instance;
- one API container/process;
- one web deployment;
- HTTPS through hosting platform/reverse proxy.

No Kubernetes requirement.

## 16. Scale path

Only scale when observed:

1. Add DB indexes/query tuning.
2. Add read caching if API profiling proves it necessary.
3. Split worker binary if ingestion competes with API latency.
4. Add a real job queue only if scheduling/retries need it.
5. Horizontal API replicas can share PostgreSQL because reads are stateless.
6. Redis is optional, not a rite of passage.

## 17. Security boundary

- Browser never sees provider credentials.
- Admin/build operations are separate from read endpoints.
- Provider payloads are treated as untrusted external input.
- SQLx parameterization is mandatory.
- Raw payload storage must avoid accidental secret persistence if providers echo credentials.

## 18. Architectural non-decisions

Deferred until evidence exists:

- Redis;
- Kafka/NATS;
- Kubernetes;
- microservices;
- CQRS/event sourcing;
- GraphQL;
- TimescaleDB;
- distributed workflow engine.
