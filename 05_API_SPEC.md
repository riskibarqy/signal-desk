# SignalDesk - REST API Specification

Version: 1.0  
Base path: `/api/v1`

## 1. Conventions

- JSON only for M1.
- Dates use `YYYY-MM-DD`.
- Machine timestamps use RFC 3339 UTC.
- Ratios are returned as decimal fractions, not display percentages.
- Currency fields include `_idr` suffix where practical.
- Null means unavailable; never substitute zero.
- API response metadata includes methodology versions where relevant.

## 2. Success envelope

```json
{
  "data": {},
  "meta": {
    "request_id": "2b5d...",
    "metric_version": "structure-v1",
    "classification_version": "class-v1"
  }
}
```

## 3. Error envelope

```json
{
  "error": {
    "code": "INVALID_DATE",
    "message": "date must use YYYY-MM-DD"
  },
  "meta": {
    "request_id": "2b5d..."
  }
}
```

Stable error codes are part of the contract. Human messages may change.

## 4. GET /health

Response:

```json
{
  "data": {
    "status": "ok",
    "database": "ok"
  },
  "meta": {
    "request_id": "..."
  }
}
```

## 5. GET /radar

Query:

- `date` required initially, or default to latest completed date later;
- `classification` optional;
- `sector` optional;
- `min_traded_value_idr` optional;
- `min_free_float` optional;
- `max_free_float` optional;
- `sort` optional;
- `order=asc|desc` optional.

Example item:

```json
{
  "symbol": "ABCD",
  "company_name": "Example Tbk",
  "trading_date": "2026-09-10",
  "close_idr": 1230,
  "return_1d": 0.042,
  "traded_value_idr": 812000000000,
  "market_cap_idr": 48000000000000,
  "free_float_ratio": 0.094,
  "free_float_mcap_idr": 4512000000000,
  "float_turnover": 0.168,
  "broker_hhi": 0.141,
  "effective_brokers": 7.09,
  "raw_active_brokers": 42,
  "cr5": 0.63,
  "foreign_flow_intensity": -0.0062,
  "classification": "concentrated_float_intense",
  "quality": "ok",
  "reason_codes": ["FLOAT_TURNOVER_HIGH", "BROKER_HHI_HIGH"]
}
```

## 6. GET /stocks/{symbol}

Query:

- `date` required for M1.

Returns:

- instrument identity;
- market daily input;
- free-float input;
- derived metrics;
- classification;
- quality details;
- provenance summary;
- broker distribution summary.

## 7. GET /stocks/{symbol}/brokers

Query:

- `date` required;
- `limit` optional, max 100.

Example:

```json
{
  "data": {
    "symbol": "ABCD",
    "trading_date": "2026-09-10",
    "brokers": [
      {
        "broker_code": "XX",
        "buy_value_idr": 100000000000,
        "sell_value_idr": 80000000000,
        "gross_value_idr": 180000000000,
        "participation_share": 0.19
      }
    ],
    "summary": {
      "hhi": 0.141,
      "effective_brokers": 7.09,
      "cr5": 0.63,
      "raw_active_brokers": 42
    }
  },
  "meta": {}
}
```

## 8. GET /compare

Query:

- `symbols=BBCA,BBRI` required;
- 2-4 unique symbols;
- `date` required.

Response contains an array with the same structural metric schema used by radar/detail.

## 9. GET /methodology

Returns:

```json
{
  "data": {
    "metric_version": "structure-v1",
    "classification_version": "class-v1",
    "metrics": [
      {
        "key": "float_turnover",
        "name": "Float Turnover",
        "formula": "traded_shares / free_float_shares",
        "unit": "ratio",
        "description": "Trading activity relative to estimated tradable shares.",
        "caveats": ["Depends on point-in-time free-float validity"]
      }
    ]
  },
  "meta": {}
}
```

## 10. GET /available-dates

Returns completed radar dates, latest first.

## 11. GET /stocks/{symbol}/structure-history

Query:

- `from` optional;
- `to` optional;
- `limit` bounded.

Only returns point-in-time valid metric dates.

## 12. POST /admin/build

Protected operator endpoint.

Request:

```json
{
  "trading_date": "2026-09-10",
  "force_refresh": false
}
```

Response:

```json
{
  "data": {
    "build_id": "uuid",
    "status": "accepted"
  },
  "meta": {
    "request_id": "uuid"
  }
}
```

If synchronous M1 implementation is simpler, the endpoint may block until completion, but the response contract must clearly expose status. Long term, prefer asynchronous build status.

## 13. GET /admin/builds/{build_id}

Returns:

- status `building|completed|failed`;
- start/end timestamps;
- symbols processed;
- provider calls;
- cache hits;
- quality rejection counts;
- error summary if failed.

## 14. Pagination

M1 radar may remain unpaginated if candidate universe is deliberately small. Instrument/history endpoints should use limit/cursor or limit/date windows when result sizes grow.

Do not invent pagination complexity where 20 rows exist.

## 15. Caching headers

Read responses may include ETag/Last-Modified later. M1 can rely on snapshot immutability and application-level browser caching.

## 16. OpenAPI

Once route shapes stabilize, generate OpenAPI from Rust types/route annotations using a suitable Axum-compatible crate. Generated OpenAPI must reflect actual response DTOs; handwritten API docs remain conceptual guidance, not a second conflicting source of truth.
