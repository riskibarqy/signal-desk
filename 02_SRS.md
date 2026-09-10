# SignalDesk - Software Requirements Specification

Version: 2.0  
Implementation baseline: Rust + Axum + PostgreSQL + SvelteKit  
Scope of this SRS: M0 Data Feasibility and M1 Structural Liquidity MVP

## 1. Purpose

This SRS defines implementable requirements for SignalDesk's first production milestone. It covers source ingestion, normalized storage, structural-liquidity calculations, snapshot generation, REST APIs, frontend behavior, data-quality handling, security, observability, and tests.

## 2. System scope

The system shall provide:

1. a data-feasibility CLI or command;
2. persistent provider ingestion;
3. normalized daily market observations;
4. free-float snapshots;
5. broker-participation observations;
6. foreign-flow observations;
7. deterministic structural metrics;
8. versioned classifications;
9. radar, detail, compare, and methodology APIs;
10. SvelteKit pages consuming only SignalDesk APIs;
11. historical completed snapshots;
12. explicit stale/missing/invalid data states.

## 3. Technology requirements

### Backend

- Rust stable toolchain.
- Axum for HTTP routing.
- Tokio for async runtime.
- Serde/serde_json for serialization.
- Reqwest for external HTTP.
- SQLx for PostgreSQL access and migrations.
- Tower/Tower HTTP for middleware.
- tracing/tracing-subscriber for structured logs.
- thiserror for typed internal errors.
- UUIDs for public/internal snapshot identifiers where appropriate.

### Frontend

- SvelteKit + TypeScript.
- Tailwind CSS.
- shadcn-svelte or equivalent accessible primitives.
- Apache ECharts for charts.
- Frontend must not call external market-data providers directly.

### Database

- PostgreSQL.
- UTC timestamps for machine times.
- IDX trading date stored separately as a SQL DATE.
- Raw provider payloads may be stored as JSONB for traceability.

## 4. High-level system context

```text
Browser
  |
  v
SvelteKit Web
  |
  | REST/JSON
  v
Rust/Axum API
  |
  +--> Application Services
  |      +--> Radar Builder
  |      +--> Structural Analytics
  |      +--> Ingestion Service
  |
  +--> PostgreSQL
  |
  +--> Provider Adapter (Sectors first)
            |
            v
       External REST API
```

## 5. Actors

### A1 - Research user

Reads radar, detail, compare, methodology, and historical data.

### A2 - Operator

Runs/monitors ingestion, refreshes a trading date, and inspects data quality.

### A3 - External data provider

Supplies market, free-float, broker, and foreign-flow data.

## 6. Domain definitions

- **Instrument**: one listed security identified by a canonical SignalDesk symbol.
- **Trading date**: IDX market date stored as `YYYY-MM-DD`.
- **Source fetch**: one provider request plus response metadata.
- **Normalized observation**: provider-independent domain data derived from one or more source fetches.
- **Metric snapshot**: immutable derived metrics for one instrument and trading date using a specific metric version.
- **Radar snapshot**: immutable set of selected instruments and classifications for one trading date and methodology version.
- **Free-float ratio**: decimal fraction from 0 to 1 inclusive.
- **Gross broker participation**: broker buy value + broker sell value.

## 7. Functional requirements

### FR-001 Health

`GET /api/v1/health` shall report API and database health without contacting the external provider.

Acceptance:

- returns HTTP 200 when API and DB are healthy;
- returns structured component status;
- does not trigger ingestion.

### FR-002 Instrument catalog

The system shall maintain a normalized instrument table containing at minimum symbol, company name, sector/industry when available, listing status, and provider identifiers.

### FR-003 Source fetch persistence

Every external fetch used for derived data shall persist:

- provider;
- endpoint category;
- symbol if applicable;
- requested trading/as-of date;
- fetch timestamp;
- HTTP status;
- provider request identifier if available;
- payload hash;
- raw JSON payload when storage policy permits;
- parse/normalization status.

### FR-004 Data-feasibility command

The application shall expose a CLI or operator command that accepts 3-5 symbols and a trading date, fetches required source data, validates units/coverage, calculates all M1 metrics, and outputs JSON plus a data-quality report.

The command shall report external call count and cache hits.

### FR-005 Cache-first provider access

Provider client calls shall first check persisted source data unless explicitly run in force-refresh mode.

### FR-006 Market observation ingestion

For each eligible instrument/trading date, persist:

- close price;
- traded shares/volume;
- traded value;
- market capitalization;
- shares outstanding when available;
- daily return when source-derived or calculate from validated closes;
- source reference(s).

### FR-007 Free-float ingestion

Persist free-float ratio with an explicit `as_of_date` and source reference.

The system shall not silently apply a current free-float ratio to older dates.

### FR-008 Broker activity ingestion

Persist broker-level buy/sell values for an instrument/trading date.

Rows shall retain the provider broker code. Broker code must not be relabeled as investor identity.

### FR-009 Foreign-flow ingestion

Persist net foreign flow for an instrument/trading date with explicit unit and source metadata.

### FR-010 Free-float market capitalization

Calculate:

`free_float_mcap = market_cap * free_float_ratio`

Validation:

- market cap > 0;
- free-float ratio in `(0, 1]`;
- dates compatible under the configured point-in-time policy.

If validation fails, result is null plus a reason code.

### FR-011 Free-float shares

Preferred:

`free_float_shares = shares_outstanding * free_float_ratio`

Fallback when shares outstanding is unavailable and market cap/close are validated:

`estimated_shares = market_cap / close`

`free_float_shares = estimated_shares * free_float_ratio`

Fallback result must set `is_estimated = true`.

### FR-012 Float turnover

Preferred:

`float_turnover = traded_shares / free_float_shares`

Secondary metric:

`float_value_turnover = traded_value / free_float_mcap`

The two metrics must remain separately named. The UI must never display one under the other's label.

### FR-013 Broker normalization

For each broker row:

`gross_i = buy_value_i + sell_value_i`

Rules:

- negative source values are invalid unless provider semantics explicitly require them;
- rows with both values zero are excluded from concentration metrics;
- units must be normalized before summing.

### FR-014 Broker HHI

For positive gross participation:

`p_i = gross_i / sum(gross_j)`

`hhi = sum(p_i ^ 2)`

Valid range: `(0, 1]`.

### FR-015 Effective broker count

`effective_brokers = 1 / hhi`

For valid observations, result must be within numeric tolerance of `[1, raw_active_broker_count]`.

### FR-016 CR5

Sort brokers by gross participation descending.

`cr5 = sum(top five p_i)`

If fewer than five brokers are active, CR5 is the sum of all active broker shares and quality metadata records the smaller broker count.

### FR-017 Foreign-flow intensity

`foreign_flow_intensity = net_foreign_flow / free_float_mcap`

This metric is signed context. It must not be interpreted as a probability or recommendation.

### FR-018 Metric quality state

Every derived metric shall include a quality state:

- `ok`;
- `estimated`;
- `stale`;
- `missing`;
- `invalid`.

Optional reason codes include:

- `FREE_FLOAT_MISSING`;
- `FREE_FLOAT_DATE_UNUSABLE`;
- `FREE_FLOAT_OUT_OF_RANGE`;
- `MARKET_CAP_MISSING`;
- `CLOSE_INVALID`;
- `BROKER_DATA_EMPTY`;
- `BROKER_TOTAL_MISMATCH`;
- `FOREIGN_FLOW_MISSING`;
- `DATE_MISMATCH`;
- `UNIT_NOT_VERIFIED`.

### FR-019 Classification engine

Classification shall be deterministic and versioned.

Initial output enum:

- `broad_liquidity`;
- `float_intense`;
- `broker_concentrated`;
- `concentrated_float_intense`;
- `thin_quiet`;
- `insufficient_data`.

Thresholds shall be configuration/data associated with a classification version, not scattered as handler constants.

### FR-020 Classification inputs

The first version may use:

- cross-sectional float-turnover percentile;
- cross-sectional broker-HHI percentile or effective-broker percentile;
- minimum absolute traded-value floor;
- minimum valid broker count;
- metric completeness.

Exact thresholds shall be frozen only after the M0 feasibility distribution is inspected.

### FR-021 Radar build

An operator build for a trading date shall:

1. acquire a single-build lease for the date/version;
2. load/update instrument universe;
3. ingest required market-wide data;
4. select eligible instruments;
5. fetch/cache per-instrument enrichment with bounded concurrency;
6. normalize and validate inputs;
7. calculate metric snapshots;
8. calculate cross-sectional percentiles;
9. classify instruments;
10. persist radar snapshot and members atomically;
11. mark build completed only after all required writes succeed.

A failed build must not replace the last completed snapshot.

### FR-022 Radar retrieval

`GET /api/v1/radar?date=YYYY-MM-DD` shall return a completed radar snapshot without calling the external provider.

### FR-023 Radar filters/sort

The frontend shall support filtering and sorting described in the PRD. Null metrics must sort consistently and remain visibly unavailable.

### FR-024 Stock detail

`GET /api/v1/stocks/{symbol}?date=YYYY-MM-DD` shall return:

- normalized market inputs;
- free-float inputs;
- structural metrics;
- classification and triggered reasons;
- broker participation rows or summarized distribution;
- foreign-flow context;
- provenance;
- quality states.

### FR-025 Compare

`GET /api/v1/compare?symbols=A,B&date=YYYY-MM-DD` shall accept 2-4 unique symbols and return a common metric schema.

### FR-026 Methodology

`GET /api/v1/methodology` shall return metric formulas, units, metric version, classification version, and plain-language caveats.

### FR-027 Admin refresh/build

`POST /api/v1/admin/build` shall be protected. It accepts a trading date and optional force-refresh flag.

The endpoint must not allow anonymous callers to consume provider quota.

### FR-028 Historical metric retrieval

Once more than one snapshot exists, `GET /api/v1/stocks/{symbol}/structure-history` shall return only dates for which the point-in-time input policy is satisfied.

## 8. Data and unit requirements

### Prices

Store IDX prices in integer IDR when provider precision permits. If source uses decimals, preserve source precision in NUMERIC.

### Shares/volume

Normalize to shares, not lots, before analytics. Raw unit must remain available in source metadata.

### Currency values

Normalize traded value, market cap, and foreign flow to IDR. Use PostgreSQL NUMERIC or BIGINT where the provider guarantees integer IDR and range is safe.

### Ratios

Store ratios as DOUBLE PRECISION for analytics plus quality/provenance. Display percentages are derived in the UI.

## 9. Database requirements

Minimum tables:

### `instruments`

- `id` UUID PK;
- `symbol` TEXT UNIQUE NOT NULL;
- `company_name` TEXT;
- `sector` TEXT NULL;
- `industry` TEXT NULL;
- `is_active` BOOLEAN;
- `provider_ids` JSONB;
- timestamps.

### `source_fetches`

- `id` UUID PK;
- `provider` TEXT;
- `endpoint_kind` TEXT;
- `instrument_id` UUID NULL;
- `requested_date` DATE NULL;
- `fetched_at` TIMESTAMPTZ;
- `http_status` INT NULL;
- `cache_key` TEXT UNIQUE;
- `payload_hash` TEXT;
- `payload` JSONB NULL;
- `parse_status` TEXT;
- `error_code` TEXT NULL.

### `market_daily`

- `instrument_id` UUID;
- `trading_date` DATE;
- `close_idr` NUMERIC;
- `traded_shares` NUMERIC;
- `traded_value_idr` NUMERIC;
- `market_cap_idr` NUMERIC;
- `shares_outstanding` NUMERIC NULL;
- `source_fetch_id` UUID;
- PK `(instrument_id, trading_date)`.

### `free_float_snapshots`

- `instrument_id` UUID;
- `as_of_date` DATE;
- `free_float_ratio` DOUBLE PRECISION;
- `source_fetch_id` UUID;
- PK `(instrument_id, as_of_date)`.

### `broker_daily`

- `instrument_id` UUID;
- `trading_date` DATE;
- `broker_code` TEXT;
- `buy_value_idr` NUMERIC;
- `sell_value_idr` NUMERIC;
- `source_fetch_id` UUID;
- PK `(instrument_id, trading_date, broker_code)`.

### `foreign_flow_daily`

- `instrument_id` UUID;
- `trading_date` DATE;
- `net_foreign_flow_idr` NUMERIC;
- `source_fetch_id` UUID;
- PK `(instrument_id, trading_date)`.

### `structure_metrics`

- `instrument_id` UUID;
- `trading_date` DATE;
- `metric_version` TEXT;
- `free_float_mcap_idr` NUMERIC NULL;
- `free_float_shares` NUMERIC NULL;
- `free_float_shares_estimated` BOOLEAN;
- `float_turnover` DOUBLE PRECISION NULL;
- `float_value_turnover` DOUBLE PRECISION NULL;
- `broker_hhi` DOUBLE PRECISION NULL;
- `effective_brokers` DOUBLE PRECISION NULL;
- `raw_active_brokers` INT NULL;
- `cr5` DOUBLE PRECISION NULL;
- `foreign_flow_intensity` DOUBLE PRECISION NULL;
- `quality` JSONB;
- PK `(instrument_id, trading_date, metric_version)`.

### `radar_snapshots`

- `id` UUID PK;
- `trading_date` DATE;
- `metric_version` TEXT;
- `classification_version` TEXT;
- `status` TEXT;
- `created_at` TIMESTAMPTZ;
- `completed_at` TIMESTAMPTZ NULL;
- `build_meta` JSONB;
- UNIQUE `(trading_date, metric_version, classification_version, status)` only if useful; completed uniqueness enforced carefully.

### `radar_members`

- `radar_id` UUID;
- `instrument_id` UUID;
- `classification` TEXT;
- `reason_codes` JSONB;
- `float_turnover_percentile` DOUBLE PRECISION NULL;
- `hhi_percentile` DOUBLE PRECISION NULL;
- PK `(radar_id, instrument_id)`.

## 10. Transaction and consistency requirements

- Radar completion shall be atomic.
- A build may write staging/working rows before completion, but readers query completed snapshots only.
- Failed builds remain inspectable but are never returned as current radar data.
- Metric calculations must be idempotent for identical source snapshots and methodology versions.
- Database migrations must be version-controlled and applied before application start or via an explicit migration command.

## 11. Concurrency requirements

- No unbounded Tokio task spawning per symbol.
- External calls use a semaphore/concurrency limit.
- One build per trading date + methodology version at a time.
- Reads continue during ingestion/build.
- Request cancellation propagates through service and database work where practical.
- Background jobs must emit a build identifier in logs.

## 12. External-provider requirements

Provider interface shall be domain-oriented, for example:

```rust
#[async_trait]
pub trait MarketDataProvider {
    async fn instruments(&self, ctx: ProviderContext) -> Result<Vec<InstrumentInput>, ProviderError>;
    async fn market_daily(&self, date: NaiveDate) -> Result<Vec<MarketDailyInput>, ProviderError>;
    async fn free_float(&self, date: Option<NaiveDate>) -> Result<Vec<FreeFloatInput>, ProviderError>;
    async fn broker_activity(&self, symbol: &str, date: NaiveDate) -> Result<Vec<BrokerActivityInput>, ProviderError>;
    async fn foreign_flow(&self, symbol: &str, date: NaiveDate) -> Result<Option<ForeignFlowInput>, ProviderError>;
}
```

Exact Rust date type may differ. The architectural requirement is the provider boundary, not this literal signature.

External requests shall include:

- timeout;
- retry only for retryable status/network failures;
- exponential backoff with jitter;
- bounded concurrency;
- request tracing;
- persistent cache lookup;
- response-size limits where practical;
- no secret values in logs.

## 13. API conventions

### Success envelope

```json
{
  "data": {},
  "meta": {
    "request_id": "uuid",
    "trading_date": "2026-09-10",
    "metric_version": "structure-v1",
    "classification_version": "class-v1"
  }
}
```

### Error envelope

```json
{
  "error": {
    "code": "RADAR_NOT_FOUND",
    "message": "No completed radar snapshot exists for this trading date"
  },
  "meta": {
    "request_id": "uuid"
  }
}
```

Frontend logic shall use stable error codes, not parse message strings.

## 14. Non-functional requirements

### Performance

Target on a normal single-instance deployment with warm database/cache:

- radar API p95 < 350 ms;
- stock detail p95 < 250 ms;
- compare p95 < 350 ms for four symbols;
- methodology p95 < 100 ms.

These are not market-data latency targets because reads must not depend on live provider calls.

### Reliability

- User reads remain available when the provider is down if completed local data exists.
- No partial completed snapshot.
- A failed refresh preserves the previous completed snapshot.
- Provider failures are recorded with retryability metadata.

### Security

- Provider API keys are backend-only environment/secrets values.
- Admin build endpoint requires authentication/secret at minimum.
- CORS is restricted in deployed environments.
- Authorization headers and secrets are redacted from logs.
- SQL is parameterized through SQLx.
- Input validation applies to symbols, dates, list sizes, and pagination.
- Request body size is limited.
- Admin operations are rate-limited.

### Observability

Structured tracing fields should include:

- request ID;
- route;
- status;
- latency;
- symbol;
- trading date;
- build ID;
- provider endpoint kind;
- cache hit/miss;
- provider latency/status;
- row counts;
- quality failure counts.

Metrics to add when useful:

- API request duration/count;
- provider call count/failure rate;
- cache hit ratio;
- build duration;
- instruments processed;
- data-quality rejection count;
- DB pool saturation.

## 15. Frontend requirements

### Radar

- table-first;
- URL-persisted date/filter state where reasonable;
- loading/empty/error/completed states;
- null displayed as `Unavailable`, not `0`;
- methodology tooltip/link for each derived column;
- data-quality badge.

### Detail

- metric cards with exact units;
- broker distribution chart;
- headline vs free-float market-cap comparison;
- provenance panel;
- classification explanation;
- historical chart only when point-in-time inputs are valid.

### Compare

- 2-4 instruments;
- same trading date;
- same metric definitions;
- no rescaling that visually hides missing values.

### Accessibility

- keyboard-usable tables/filters;
- semantic headings;
- sufficient contrast;
- chart data has textual/tabular fallback for important values.

## 16. Error handling

Recommended HTTP mapping:

- invalid input: 400;
- admin unauthenticated: 401;
- admin forbidden: 403;
- snapshot/instrument not found: 404;
- build already running: 409;
- explicit provider refresh failed: 502/503;
- unexpected internal failure: 500.

Read endpoints must not return 5xx merely because the provider is down when local data exists.

## 17. Testing requirements

### Unit tests

Required for:

- free-float market cap;
- estimated shares;
- float turnover;
- value turnover;
- HHI;
- effective broker count;
- CR5;
- foreign-flow intensity;
- quality propagation;
- classification boundaries;
- unit conversion helpers.

### Invariant/property tests

At minimum:

- `0 < HHI <= 1` for valid positive broker sets;
- `1 <= effective_brokers <= raw_active_broker_count` within tolerance;
- `0 < CR5 <= 1`;
- free-float market cap <= headline market cap for ratio <= 1;
- deterministic metrics for identical normalized inputs;
- permutation of broker rows does not change HHI/CR5.

### Integration tests

- migrations on temporary PostgreSQL database;
- repository upsert/idempotency;
- provider fixture -> normalization -> metric snapshot;
- failed radar build leaves prior completed snapshot readable;
- concurrent build lease prevents duplicate build;
- API uses DB and does not call provider on read path.

### Frontend tests

- radar states;
- filters/sorting;
- null handling;
- detail rendering;
- compare selection limits;
- methodology navigation.

### End-to-end smoke test

1. Start PostgreSQL, API, and web.
2. Load recorded provider fixtures or local seed data.
3. Build one radar date.
4. Open radar.
5. Open stock detail.
6. Compare stocks.
7. Disable external network/provider configuration.
8. Repeat read flow successfully.

## 18. M0 acceptance criteria

Data feasibility passes only when:

1. free-float coverage is usable for chosen samples;
2. market-cap, price, volume, and traded-value units are manually verified;
3. volume is confirmed as shares or converted correctly from lots;
4. broker rows produce plausible totals and concentration values;
5. foreign-flow sign and currency units are verified;
6. source dates align sufficiently for the intended metric;
7. provider quota/cost behavior is understood enough to budget ingestion;
8. all unit assumptions are documented in a fixture note.

If one of these fails, the affected metric is removed or marked unavailable. Data is not coerced until the specification becomes true.

## 19. M1 acceptance criteria

M1 is accepted when:

1. real or recorded source data can build a completed radar snapshot;
2. at least 10 securities display valid structural metrics;
3. all core formulas have unit and invariant tests;
4. API reads are database-backed;
5. detail and compare use common metric definitions;
6. metric/classification versions are persisted;
7. historical snapshots are immutable once completed;
8. missing/invalid values remain explicit;
9. failed builds do not replace completed data;
10. provider key is absent from browser artifacts and logs;
11. application can start through documented Docker/local commands;
12. methodology is visible from the UI.

## 20. Change control

M0/M1 requirements may change only when:

- actual source payloads invalidate a data assumption;
- correctness/security requires a change;
- implementation complexity is materially higher than expected and scope reduction preserves the core product.

New product features belong in later milestones, not in M1 by enthusiasm alone.
