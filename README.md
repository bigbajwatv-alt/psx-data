# PSX Data — URL & Schema Reference

Public Pakistan Stock Exchange data published from this project. Two layers:

1. **GitHub Pages files** (this project's pre-built JSON, crawler-friendly).
2. **PSX DPS endpoints** (the upstream JSON/HTML, also crawler-friendly — no
   robots.txt on `dps.psx.com.pk`).

Both layers are safe to fetch from Claude.ai, scripts, or anything else.

---

## 1. GitHub Pages URLs (pre-built JSON)

Base: `https://bigbajwatv-alt.github.io/psx-data/`

| File | Refresh | Approx size | Source |
| --- | --- | --- | --- |
| [`latest_snapshot.json`](#latest_snapshotjson) | every ~5 min, market hours | ~8 KB | psx_api.fetch_latest for watchlist + indices |
| [`daily_summary.json`](#daily_summaryjson) | once per trading day (market close) | ~100 KB | `/market-watch` parsed |
| [`indices_eod.json`](#indices_eodjson) | once per trading day | ~2 KB | `/timeseries/eod/{INDEX}` for 7 indices |
| [`symbols.json`](#symbolsjson) | once per trading day | ~135 KB | passthrough of `/symbols` |
| [`sectors.json`](#sectorsjson) | once per trading day | ~25 KB | derived from `/symbols` |

### `latest_snapshot.json`

Watchlist + indices, enriched with day high/low from `POST /historical`.

```json
{
  "timestamp": "2026-05-21T20:30:12+05:00",
  "source": "dps.psx.com.pk",
  "market_status": "open|closed|pre_open",
  "watchlist": {
    "OGDC": {
      "symbol": "OGDC",
      "close": 325.82,
      "prev_close": 318.52,
      "change": 7.30,
      "change_pct": 2.29,
      "volume": 5129543,
      "open": 322.0,
      "day_high": 326.8,
      "day_low": 322.0,
      "as_of": "2026-05-21T11:00:00Z"
    }
  },
  "indices": { "KSE100": { /* same shape */ } },
  "errors": []
}
```

### `daily_summary.json`

Every listed security in one compact array, taken at market close.

```json
{
  "as_of": "2026-05-21T20:40:00+05:00",
  "source": "https://dps.psx.com.pk/market-watch",
  "count": 490,
  "schema": {
    "sym": "ticker",
    "sec_code": "PSX sector code (numeric)",
    "listed": "comma-separated indices the ticker belongs to",
    "ldcp": "last day close price",
    "open": "today's open",
    "high": "today's high",
    "low": "today's low",
    "close": "today's close (=current during open hours)",
    "change": "close - ldcp",
    "pct": "percent change",
    "volume": "shares traded today"
  },
  "stocks": [
    {
      "sym": "HASCOL", "sec_code": "0821", "listed": "ALLSHR",
      "ldcp": 23.0, "open": 23.44, "high": 24.39, "low": 23.28,
      "close": 23.97, "change": 0.97, "pct": 4.217, "volume": 57001579
    }
  ]
}
```

### `indices_eod.json`

Latest EOD bar for all 7 tracked indices.

```json
{
  "as_of": "2026-05-21T20:40:00+05:00",
  "source": "https://dps.psx.com.pk/timeseries/eod/{INDEX}",
  "indices": {
    "KSE100": {
      "ts": 1779361200,
      "close": 168514.44,
      "prev_close": 164831.42,
      "change": 3683.02,
      "pct": 2.234,
      "open": 166638.563,
      "volume": 270584554
    }
  },
  "errors": []
}
```

### `symbols.json`

Full PSX ticker list with sector and instrument-type flags.

```json
{
  "as_of": "2026-05-21T20:40:00+05:00",
  "source": "https://dps.psx.com.pk/symbols",
  "count": 1048,
  "schema": { "symbol": "ticker", "name": "company / instrument name",
              "sectorName": "human-readable sector",
              "isETF": "true if listed as ETF",
              "isDebt": "true if debt instrument" },
  "symbols": [
    {"symbol":"OGDC","name":"Oil & Gas Development Co. Ltd.","sectorName":"OIL & GAS EXPLORATION COMPANIES","isETF":false,"isDebt":false}
  ]
}
```

### `sectors.json`

Sector → tickers index, derived from `symbols.json` (no extra request).

```json
{
  "as_of": "2026-05-21T20:40:00+05:00",
  "source": "derived from /symbols",
  "count": 40,
  "sectors": {
    "COMMERCIAL BANKS": { "count": 22, "symbols": ["AKBL", "BAFL", "BAHL", "..."] }
  }
}
```

---

## 2. PSX DPS Endpoints (upstream)

Base: `https://dps.psx.com.pk/`

### Confirmed working — JSON

#### `GET /timeseries/eod/{SYMBOL}`

Up to 5 years of end-of-day bars for any listed security or index.

- **Accepts:** stock tickers (e.g. `HBL`, `OGDC`) and index tickers (`KSE100`, `KSE30`, `KMI30`, `ALLSHR`, `BKTI`, `OGTI`, `KMIALLSHR`).
- **Returns:**
  ```json
  { "status": 1, "message": "",
    "data": [[unix_ts, close, volume, open], ...]
  }
  ```
- **Sort order:** descending (latest row first).
- Empirically verified by computing day-over-day change against a known external source — schema matches exactly.

Example: `https://dps.psx.com.pk/timeseries/eod/HBL`

#### `GET /timeseries/int/{SYMBOL}`

Intraday tick series.

- **Returns:**
  ```json
  { "status": 1, "message": "",
    "data": [[unix_ts, last_price, volume], ...]
  }
  ```
- **Sort order:** descending. Covers only the current trading day.

Example: `https://dps.psx.com.pk/timeseries/int/HBL`

#### `GET /timeseries/1min/{SYMBOL}`

**Alias of `/timeseries/eod/`** — returns the same bytes. Don't bother.

#### `GET /symbols`

Full bulk ticker list.

- **Returns:** JSON array of `{symbol, name, sectorName, isETF, isDebt}` objects.
- ~1000 records, ~135 KB.
- Same payload that powers our `symbols.json`.

Example: `https://dps.psx.com.pk/symbols`

### Confirmed working — HTML (needs parsing)

#### `POST /historical` (form: `month`, `year`, `symbol`)

The **only** endpoint that exposes day HIGH and LOW.

- **Request:** `application/x-www-form-urlencoded` body `month=5&year=2026&symbol=HBL`.
- **Returns:** HTML page containing one `<table>` with columns:
  `DATE, OPEN, HIGH, LOW, CLOSE, VOLUME`. One row per trading day in the
  requested month, latest first.
- **Used by:** `fetch_day_ohlc()` in `psx_api.py` to enrich `latest_snapshot.json`
  with `day_high` / `day_low`.

Example request (curl):
```bash
curl -X POST https://dps.psx.com.pk/historical \
  -d "month=5&year=2026&symbol=HBL" \
  -H "Content-Type: application/x-www-form-urlencoded"
```

#### `GET /market-watch`

All listed securities' current OHLCV, all in one HTML page (~470 KB).

- **Returns:** HTML with a single `<table>`; each `<tr>` has positional `<td>`
  cells where every numeric cell carries a `data-order="…"` attribute
  containing the raw value (no comma stripping needed).
- **Columns (in order):** symbol, sector_code, listed_in_indices, LDCP, OPEN,
  HIGH, LOW, CURRENT, CHANGE, %CHANGE, VOLUME.
- **Used by:** `daily_publisher.py` to build `daily_summary.json`.

Example: `https://dps.psx.com.pk/market-watch`

#### `GET /company/{SYMBOL}`

Company profile page. HTML, JS-rendered for sub-tabs. Not currently parsed.

Example: `https://dps.psx.com.pk/company/HBL`

#### `GET /indices`

PSX index landing page. HTML; uses live data via embedded JS. Not directly
useful — bundle the indices' EOD via `/timeseries/eod/{INDEX}` instead.

### Confirmed 404 / not endpoints

- `GET /historical/{SYMBOL}` — 404 (use `POST /historical` instead).
- `GET /symbol/{SYMBOL}` — 404 (use `GET /symbols` for the bulk list).
- `GET /quote/{SYMBOL}` — 404 (use `GET /timeseries/eod/{SYMBOL}` for latest).

---

## 3. Refresh cadence

| File | Trigger | Mechanism |
| --- | --- | --- |
| `latest_snapshot.json` | every `POLL_INTERVAL_SECONDS` (default 300s) during PKT market hours | `snapshot.py --loop` |
| `daily_summary.json` + `indices_eod.json` + `symbols.json` + `sectors.json` | once per trading day, after `market_status` becomes `closed` | `daily_publisher.publish_if_needed()` called by the loop on every cycle, idempotent via `data/last_daily_run.txt` |

Daily files are pushed in a **single Git commit** via the Trees API for clean
history.

## 4. Suggested usage patterns

- **"What's KSE100 doing right now?"** → `latest_snapshot.json` → `indices.KSE100`
- **"Find biggest gainers across all PSX today."** → `daily_summary.json` → sort by `pct` desc
- **"Which stocks are in the Cement sector?"** → `sectors.json` → `sectors["CEMENT"].symbols`
- **"Historical OHLC for ENGRO last 5 years."** → `dps.psx.com.pk/timeseries/eod/ENGRO` (direct, no mirror needed)
- **"Intraday tape for HBL today."** → `dps.psx.com.pk/timeseries/int/HBL` (direct)

## 5. Robots and crawler-friendliness

- `dps.psx.com.pk` — no `robots.txt`, fully crawler-accessible.
- `bigbajwatv-alt.github.io` — GitHub Pages default policy is permissive.
- `gist.githubusercontent.com` — **disallows crawlers** (gist URL only works
  for direct fetches with overridden User-Agent, not for AI tools that
  respect robots.txt).
