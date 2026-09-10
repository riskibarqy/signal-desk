# SignalDesk - Data & Analytics Specification

Version: 1.0  
Scope: Structural Liquidity M0/M1

## 1. Purpose

This document defines source units, normalized fields, formulas, invariants, quality states, classifications, and point-in-time rules. This is the reference for analytics tests.

## 2. Canonical units

- Price: IDR per share.
- Volume: shares. Convert lots to shares before analytics.
- Traded value: IDR.
- Market capitalization: IDR.
- Foreign flow: signed IDR net flow.
- Free float: ratio in `[0,1]` internally.
- Percent display: ratio multiplied by 100 only at presentation.
- Trading date: IDX local market date.

Every provider field must have its unit verified before normalization.

## 3. Point-in-time rule

A derived metric for date `D` may use a free-float observation only when SignalDesk's validity policy establishes that the free-float snapshot is valid for `D`.

Default conservative policy for M1:

- exact same-date free-float snapshot: valid;
- explicitly effective-dated snapshot covering D: valid;
- current snapshot with unknown historical validity: invalid for historical D.

Never backfill today's free float over years of old price data merely because the join is convenient.

## 4. Free-float market cap

```text
free_float_mcap = market_cap * free_float_ratio
```

Required:

- market cap > 0;
- free float in `(0,1]`;
- point-in-time rule satisfied.

Invariant:

`free_float_mcap <= market_cap` when ratio <= 1.

## 5. Free-float shares

Preferred:

```text
free_float_shares = shares_outstanding * free_float_ratio
```

Fallback estimate:

```text
estimated_shares = market_cap / close
free_float_shares = estimated_shares * free_float_ratio
```

Fallback must be flagged because market cap can be rounded or based on a slightly different share count.

## 6. Float turnover

Share-based preferred metric:

```text
float_turnover = traded_shares / free_float_shares
```

Interpretation: fraction of estimated tradable share pool represented by that day's traded shares.

Value-based companion metric:

```text
float_value_turnover = traded_value / free_float_mcap
```

Do not claim these two are identical. Price dispersion and intraday variation can make them differ.

## 7. Broker gross participation

For each broker `i`:

```text
gross_i = buy_value_i + sell_value_i
```

Total gross participation:

```text
G = sum(gross_i)
```

Participation share:

```text
p_i = gross_i / G
```

The metric measures distribution across broker codes, not beneficial owners.

## 8. HHI

```text
HHI = sum(p_i^2)
```

Properties:

- approaches 1 when one broker dominates;
- decreases as participation becomes more evenly distributed;
- valid range `(0,1]` for positive gross participation.

## 9. Effective broker count

```text
effective_brokers = 1 / HHI
```

Interpretation: number of equally sized brokers that would produce the observed HHI.

Invariant:

```text
1 <= effective_brokers <= raw_active_broker_count
```

within floating-point tolerance.

## 10. CR5

Sort brokers by `p_i` descending:

```text
CR5 = p_1 + p_2 + p_3 + p_4 + p_5
```

If fewer than five brokers are active, use all active brokers and retain raw active broker count in metadata.

## 11. Foreign-flow intensity

```text
foreign_flow_intensity = net_foreign_flow / free_float_mcap
```

Signed value:

- positive = net foreign inflow under source semantics;
- negative = net foreign outflow;
- zero = balanced/no net flow.

It is context only, not a future-return prediction.

## 12. Optional contextual metrics

Useful additions after M1 stabilizes:

### Broker top-1 concentration

```text
CR1 = max(p_i)
```

### Broker entropy

```text
entropy = -sum(p_i * ln(p_i))
```

Only add if it improves interpretation beyond HHI/effective brokers.

### Relative traded value

```text
relative_traded_value = traded_value / rolling_median_traded_value
```

Requires valid historical daily data and enough lookback.

## 13. Classification methodology

Do not classify from one hard-coded absolute threshold alone. Use a combination of absolute minimum liquidity floors and cross-sectional percentiles for the trading date.

Inputs:

- float-turnover percentile;
- HHI percentile;
- effective-broker percentile as explanatory companion;
- traded-value floor;
- broker-count minimum;
- metric completeness.

Candidate initial rules after M0 validation:

```text
if mandatory data missing:
    insufficient_data
else if traded_value below minimum floor or active_brokers below minimum:
    thin_quiet
else if float_turnover_pct >= FLOAT_HIGH and hhi_pct >= HHI_HIGH:
    concentrated_float_intense
else if float_turnover_pct >= FLOAT_HIGH:
    float_intense
else if hhi_pct >= HHI_HIGH:
    broker_concentrated
else:
    broad_liquidity
```

`FLOAT_HIGH`, `HHI_HIGH`, and absolute floors are stored with `classification_version`.

## 14. Percentile calculation

Percentiles shall be calculated only across eligible instruments with valid values for the metric on that date.

Store:

- eligible sample size;
- percentile method/version;
- excluded count and major exclusion reasons.

If sample size is too small, classification using that percentile is invalid.

## 15. Quality model

Each metric has:

```json
{
  "state": "ok|estimated|stale|missing|invalid",
  "reason_codes": [],
  "source_dates": [],
  "calculation_version": "structure-v1"
}
```

Examples:

- market cap exists, free float missing -> free-float metrics `missing`;
- shares outstanding missing but estimate possible -> free-float shares `estimated`;
- broker response empty -> broker metrics `missing`;
- HHI > 1 due bad units/normalization -> broker metrics `invalid`;
- current free float joined to historical date without validity -> `invalid`, not `estimated`.

## 16. Reconciliation checks

### Broker

- no negative normalized gross values;
- total gross > 0;
- all participation shares sum approximately to 1;
- HHI and CR5 ranges valid;
- investigate provider-level buy/sell totals if documented aggregates are available.

### Market

- close > 0 for traded instruments;
- traded shares >= 0;
- traded value >= 0;
- market cap > 0 where required;
- date matches requested trading date.

### Free float

- ratio > 0 and <= 1;
- as-of/effective date policy satisfied.

### Foreign flow

- currency and sign semantics verified;
- date aligns with market observation.

## 17. Versioning

At minimum persist:

- `normalization_version`;
- `metric_version`;
- `classification_version`.

A formula change creates a new metric version. Do not silently recalculate old rows under a new definition without preserving version identity.

## 18. Future analytics

Later modules may add:

### Fundamentals

- revenue growth;
- earnings growth;
- margins;
- ROE/ROA;
- leverage;
- free cash flow.

### Valuation

- P/E;
- P/B;
- EV/EBITDA;
- FCF yield;
- dividend yield;
- sector/peer/historical comparison.

### Market behavior

- market-relative return;
- sector-relative return;
- rolling volatility;
- abnormal volume;
- momentum windows.

### Events

- corporate action proximity;
- filing/news timeline;
- insider/major holder changes.

These dimensions remain separate rather than becoming one opaque universal score.
