# SignalDesk - Product Requirements Document

Version: 5.0  
Status: Product direction frozen; implementation may change only when source data proves an assumption wrong  
Product type: Personal long-term market research application  
Primary market: Indonesia Stock Exchange (IDX)

## 1. Product summary

SignalDesk is an evidence-first equity research workspace for Indonesia-listed stocks. Its first production module is **Structural Liquidity**, which explains whether a stock's headline trading activity is broad and well-distributed or concentrated around a small free float and a narrow set of broker participants.

The long-term product expands from this first module into a unified research terminal covering:

- structural liquidity and market participation;
- broker and foreign flow context;
- company fundamentals;
- valuation and peer comparison;
- market behavior and relative strength;
- corporate events and filings;
- watchlists, notes, and alerts;
- later, an AI research assistant grounded only in stored source data.

SignalDesk is not a stock-picking oracle. Its job is to reduce research noise and make the evidence behind a market observation easier to inspect.

## 2. Core product question

> Does a stock's headline trading activity reflect broad liquidity, or is that activity occurring through a narrow free float and concentrated broker participation?

This question is the first module because it is useful, differentiable, explainable, and can be derived from data already available from market-data providers such as Sectors.

## 3. Product goals

1. Give a researcher a fast post-market view of structural liquidity across IDX securities.
2. Normalize trading activity by free float instead of relying on raw volume or headline market capitalization alone.
3. Quantify broker participation concentration with transparent formulas.
4. Show foreign participation as context, not as a directional trading signal.
5. Keep raw source data, normalized data, and derived metrics clearly separated.
6. Preserve historical snapshots so methodology can be reproduced later.
7. Build the product as a long-lived personal research system rather than a disposable demo.
8. Keep the architecture simple enough for one developer to operate.

## 4. Product principles

### 4.1 Evidence before conclusions

Every derived metric must be traceable to source inputs and an as-of date.

### 4.2 No single magic score

SignalDesk should not collapse unrelated market dimensions into one opaque score. Free float, turnover, concentration, foreign flow, fundamentals, valuation, and market behavior remain separate dimensions.

### 4.3 Missing data must remain missing

A missing value is not zero. A stale snapshot is not current. An estimate must be labeled as an estimate.

### 4.4 Avoid causal overclaiming

Broker concentration, foreign flow, news, and corporate actions can coincide with a price move. SignalDesk must not claim that one observed variable caused the move unless the relationship is directly established.

### 4.5 No fake precision

Thresholds and classifications should be interpretable and versioned. The UI must avoid decimal-heavy pseudo-scientific scoring that suggests more certainty than the data supports.

### 4.6 Local data first

User-facing reads should come from SignalDesk's database. External providers are ingestion sources, not live dependencies for every page view.

## 5. Target users

### Primary user

An active IDX researcher who understands basic price/volume data and wants to investigate the structure underneath headline liquidity.

### Secondary users

- individual equity analysts;
- quant-minded retail investors;
- finance writers/content creators;
- developers learning market microstructure and data engineering;
- future personal use for watchlists and structured research notes.

## 6. Jobs to be done

- When a stock trades heavily, help me determine whether the activity is broad or concentrated.
- When two stocks have similar trading value, help me compare how much of their actual free float changed hands.
- When broker participation is concentrated, help me quantify that concentration rather than eyeballing a broker table.
- When foreign flow is large, show its scale relative to the tradable company value instead of showing a raw number alone.
- When data is incomplete, tell me exactly what cannot be calculated.
- When I return later, preserve historical snapshots so I can compare structural behavior over time.

## 7. Initial product module: Structural Liquidity

### 7.1 Market Structure Radar

A table-first page covering a selected trading date.

Minimum columns:

- symbol;
- company name;
- close and daily return;
- traded value;
- headline market capitalization;
- free-float percentage;
- free-float market capitalization;
- float turnover;
- broker HHI;
- effective broker count;
- CR5;
- foreign-flow intensity;
- structural classification;
- data-quality status;
- as-of date.

Filters:

- symbol/company;
- sector;
- minimum traded value;
- market-cap range;
- free-float range;
- structural classification;
- data-quality status.

Sorting:

- float turnover;
- HHI;
- effective brokers;
- CR5;
- foreign-flow intensity;
- traded value;
- free-float percentage.

### 7.2 Stock Structure Detail

The detail page shall show:

- headline market cap vs free-float market cap;
- traded value and traded shares;
- free-float percentage;
- float turnover, with formula variant and units;
- broker participation distribution;
- HHI, effective brokers, and CR5;
- foreign-flow intensity;
- structural classification explanation;
- source provenance and quality flags;
- historical structural metrics once enough snapshots exist.

### 7.3 Compare

Compare 2-4 securities using identical metric definitions and one common trading date.

### 7.4 Methodology

A methodology page must explain every derived metric in plain language, including formula, units, caveats, and version.

## 8. Structural classifications

Initial classes:

- **Broad Liquidity** - participation is relatively broad and float turnover is not unusually high.
- **Float-Intense** - trading activity is unusually high relative to estimated free float.
- **Broker-Concentrated** - a relatively small effective set of brokers accounts for a large share of gross participation.
- **Concentrated + Float-Intense** - both conditions are elevated.
- **Thin / Quiet** - activity is below the minimum liquidity floor or participation is too sparse to interpret usefully.
- **Insufficient Data** - mandatory inputs are missing, stale, invalid, or fail reconciliation.

These classes are research descriptions. They do not mean bullish, bearish, safe, unsafe, compliant, manipulated, or investable.

## 9. Data sources

### Required for M1

The provider abstraction must support these logical source categories:

- instrument/company master;
- daily price and volume;
- daily traded value;
- market capitalization;
- free-float snapshot;
- per-symbol broker activity;
- broker registry/metadata when available;
- foreign-flow data.

Sectors is the first provider. Provider-specific DTOs must not leak into domain calculations.

### Later sources

Potential later ingestion categories:

- financial statements;
- valuation ratios;
- corporate actions;
- insider/major shareholder filings;
- news;
- suspensions/market notices;
- macro/sector benchmarks.

## 10. Product scope by milestone

### M0 - Data feasibility

Purpose: prove units, coverage, dates, and formula feasibility against real data.

Output: CLI/JSON for 3-5 deliberately different stocks.

### M1 - Structural Liquidity MVP

- ingestion and persistent cache;
- PostgreSQL schema;
- core structural formulas;
- radar;
- detail;
- compare;
- methodology/provenance;
- manual/operator refresh;
- historical storage of created snapshots.

### M2 - Research Page

- historical price/value chart;
- structural metric history;
- notes;
- saved comparisons;
- watchlist.

### M3 - Fundamentals and Valuation

- revenue, earnings, margins, ROE/ROA, leverage, cash flow;
- P/E, P/B, EV/EBITDA where available;
- historical and peer comparison;
- no arbitrary all-in-one quality score.

### M4 - Market Behavior and Events

- market/sector-relative returns;
- abnormal volume;
- volatility;
- news/corporate actions/filings timeline;
- event coincidence, not unsupported causality.

### M5 - Alerts

- user-defined thresholds;
- daily structural changes;
- watchlist event notifications;
- rate-limited background evaluation.

### M6 - Grounded Research Assistant

An assistant may summarize SignalDesk's stored evidence, compare companies, and explain metrics. It must cite stored facts and cannot silently invent missing financial data.

## 11. Non-goals

For the foreseeable product:

- no automated order execution;
- no direct brokerage integration in core scope;
- no target prices;
- no guaranteed return prediction;
- no manipulation/gorengan/bandar accusations;
- no beneficial-owner inference from broker codes;
- no recreation of proprietary index/regulator methodology;
- no HFT or tick-level execution system;
- no microservice split without a concrete scaling or ownership reason.

## 12. User experience principles

- Desktop-first research workspace, responsive on mobile.
- Tables before decorative dashboards.
- Charts only when they make comparison easier.
- Every metric should have a tooltip/methodology link.
- Quality states should be visible without opening developer tools.
- Dates and units must be explicit.
- Null is displayed as unavailable, never as zero.
- Structural classification must show the reasons that triggered it.

## 13. Technology baseline

### Backend

- Rust;
- Axum;
- Tokio;
- Serde;
- Reqwest;
- SQLx;
- PostgreSQL;
- Tower/Tower HTTP;
- Tracing;
- Thiserror.

### Frontend

- SvelteKit;
- TypeScript;
- Tailwind CSS;
- shadcn-svelte or equivalent minimal component layer;
- Apache ECharts;
- optional TanStack Table if table complexity warrants it.

### Deployment

- Docker for reproducibility;
- PostgreSQL as the system of record;
- one API process initially;
- background ingestion in the same binary/process initially, separable later;
- no Redis until measured need exists.

## 14. Why Rust + SvelteKit

This product is personal and long-term, so the goal is not the absolute shortest path to a three-week demo. Rust is justified because it provides a strong type system, predictable runtime behavior, excellent async/networking support, and a useful real-world learning project. SvelteKit keeps the frontend comparatively small and fast to iterate.

The tradeoff is that Rust development is slower than Go for a developer already fluent in Go. The product accepts that cost because learning and long-term maintainability are part of the objective.

## 15. Success criteria for M1

M1 is successful when:

1. 3-5-stock feasibility spike proves units and source coverage.
2. The system can build a daily structural snapshot from real data.
3. Radar displays at least 10 real IDX securities with valid metrics.
4. Core formulas are unit-tested and deterministic.
5. Detail and compare use the same definitions as radar.
6. Source data is stored and user-facing reads do not depend on a live provider call.
7. Failed ingestion cannot corrupt or replace the previous completed snapshot.
8. Missing/invalid data remains explicit.
9. The application can be rebuilt locally from documented commands.
10. The methodology page is sufficient for another developer to reproduce the calculations.

## 16. Product risks

### Data coverage risk

Free-float or broker data may not cover every security or every date. Mitigation: eligibility rules and explicit missing-data states.

### Historical-point-in-time risk

A current free-float snapshot must not be applied backward to old dates unless point-in-time validity is known. Mitigation: store free-float observations by as-of date and only calculate historical metrics from known valid snapshots.

### Unit mismatch risk

Provider fields may use lots, shares, percentages, fractions, or currency scaling. Mitigation: M0 blocks product work until units are manually verified against sample securities.

### Interpretation risk

Users may interpret concentrated broker activity as manipulation or future price direction. Mitigation: neutral language and visible methodology caveats.

### Scope risk

A personal project can expand forever. Mitigation: milestones are sequential; M2 does not begin until M1 acceptance criteria pass.

## 17. Product decision log

- Generic stock score: rejected because it creates false precision and weak differentiation.
- Stock prediction engine: rejected as an initial product because validation would dominate the project and still not guarantee useful predictions.
- Pure anomaly detector: retained as a future module, not the core differentiator.
- Structural liquidity: accepted as the first module.
- Rust + SvelteKit: accepted for a long-term personal project.
- PostgreSQL: accepted over SQLite because historical ingestion, queries, snapshots, and future modules justify a durable database from the start.
- Redis: explicitly deferred.
- Microservices: explicitly deferred.

**Product status:** frozen for M0/M1. New features belong in later milestones unless real source data invalidates the design.
