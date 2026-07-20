# Trading Assistant — Low-Level Design

## Module / Package Breakdown

| File | Runtime | Role |
|---|---|---|
| `server.js` (~1629 lines) | Node | Express app; Upstox OAuth + proxy helpers (`fetchUpstox`, `getUpstoxOptionChain`); NSE session (`initNSESession`, `fetchNSE`) and Puppeteer stealth (`initBrowser`, `fetchNSEWithBrowser`); Yahoo fetch/parse; `SilverPredictionEngine`; all `/api/*` and `/upstox/*` routes; static serving; `getCachedOrFetch` cache. |
| `indicators.js` (~614 lines) | Node **and** browser | `TechnicalIndicators` class + a singleton `indicators`; conditional `module.exports`. Pure functions over price/candle arrays. |
| `scanner.js` (~624 lines) | Browser | `TradingScanner`: intraday + BTST scanners, signal generation, paper-trade lifecycle, daily stats/limits, `localStorage` persistence. Singleton `scanner`. |
| `api.js` (~577 lines) | Browser | `APIService`: backend client, 1.5s cache, health-check backend detection, and the full simulation fallback. Singleton `apiService`. |
| `app.js` (~993 lines) | Browser | `TradingApp`: DOM wiring, tabs, rendering, settings modal, toasts, keyboard shortcuts, paper-trade UI. Bootstrapped on `DOMContentLoaded`. |

Load order in `index.html`: `indicators.js` → `api.js` → `scanner.js` → `app.js`, so each singleton is defined before its consumers.

## Key Backend Routes

| Method | Path | Purpose |
|---|---|---|
| GET | `/upstox/login` | Return the Upstox authorization-dialog URL. |
| POST | `/upstox/auth` | Exchange an auth `code` (JSON body) for an access token. |
| GET | `/callback` | OAuth redirect target; exchanges `?code=` for a token, stores it in memory, returns an HTML success page. |
| GET | `/upstox/status` | Report connected/expired state of the in-memory token. |
| GET | `/upstox/option-chain/:symbol` | Raw Upstox option chain for NIFTY/BANKNIFTY (requires token). |
| GET | `/api/health` | Liveness probe used by the client to decide live-vs-simulation. |
| GET | `/api/indices` | NIFTY / BANKNIFTY / VIX (NSE, Yahoo fallback) + SENSEX (Yahoo). |
| GET | `/api/nifty`, `/api/banknifty`, `/api/sensex` | Index quote **with OHLCV candles** (Yahoo primary, NSE fallback). |
| GET | `/api/vix` | INDIA VIX (NSE, Yahoo fallback). |
| GET | `/api/option-chain/:symbol` | Normalized option chain: Upstox → NSE spot → simulated chain. |
| GET | `/api/option/:symbol/:strike/:type` | Single option (CE/PE) detail for a strike. |
| GET | `/api/stocks` | Quotes for the BTST stock universe (NSE NIFTY-50 constituents). |
| GET | `/api/stock/:symbol` | Single stock quote + history (Yahoo `<SYM>.NS`). |
| GET | `/api/market-status` | NSE market status, with an IST clock-based fallback. |
| GET | `/api/silver/all` | All silver instruments (COMEX + 4 Indian ETFs) with predictions. |
| GET | `/api/silver/:symbol` | Single-instrument silver prediction. |
| GET | `/api/silver` | Backward-compatible COMEX-only silver response. |
| GET | `*` (non-`/api`) | SPA catch-all → serves `index.html`. |

Server listens on **port 3000** and also serves the static frontend from the project root.

## Core Data Structures (in-memory)

- **Candle arrays** — `{ timestamp, open, high, low, close, volume }[]`, produced by `parseYahooData`/`parseOHLCV` (sliced to the last ~100) and consumed by every indicator. The client keeps a rolling `priceHistory[symbol]` of these.
- **Normalized option chain** — `{ symbol, spotPrice, atmStrike, expiry, optionChain[], pcr, maxPain, atmIV, totalCallOI, totalPutOI, simulated }`, where each row is `{ strike, isATM, callData, putData }` and each side carries `{ ltp, change, iv, delta, oi, oiChange, volume, ... }`. Upstox, NSE, and simulated sources are all coerced into this one shape.
- **Cache map** — `Map<string, { data, timestamp }>` on the server (and a parallel one in `APIService`); TTL enforced per lookup. Silver engine has its own `Map` with a 15s TTL and a `lastFetchTime` rate-limiter.
- **`UPSTOX_CONFIG`** — `{ apiKey, apiSecret, redirectUri, accessToken, tokenExpiry }`. `accessToken`/`tokenExpiry` start `null` and are populated by the OAuth exchange; `/upstox/status` compares `Date.now()` to `tokenExpiry`. (See Security notes on the credential fields.)
- **Signal object** — emitted by the scanner: `{ symbol, strike, optionType, instrument, entryLow/High, targetLow/High, stopLoss, quantity, lots, lotSize, confidence, bias, reasoning, indicators, optionData }`.
- **Paper-trade / daily stats** — `activeTrade` (entry, current, target, SL, live P&L, status) and `dailyStats` `{ pnl, trades, wins, losses, capitalInTrade }`, persisted date-stamped to `localStorage`.

## Key Logic / Algorithms

### `TechnicalIndicators` (`indicators.js`)
- **RSI(14)** — Wilder smoothing over the price series; returns 50 on insufficient data; `getRSISignal` maps to overbought/oversold/bullish/bearish bands.
- **EMA** — standard `2/(n+1)` multiplier; `getEMASignal` detects 9/21 crossovers and price-vs-EMA position.
- **VWAP** — cumulative (typical-price × volume) / cumulative volume; `getVWAPSignal` scores deviation.
- **SuperTrend** — ATR(10) × multiplier(3) bands around HL2 to classify bullish/bearish/neutral.
- **Volume analysis** — current vs 20-period average, flags spikes.
- **Candlestick patterns** — engulfing, hammer/inverted hammer, shooting star, doji, morning/evening star, three white soldiers / three black crows; each with a confidence score, returning the highest-confidence match.
- **Pivot points** — classic pivot with R1/R2/S1/S2 (in the silver engine).
- **`calculateComprehensiveSignal`** — the composite: weighted bullish/bearish accumulation (**EMA 30, RSI 25, VWAP 20, SuperTrend 15, volume 10, candlestick 10**, plus a price-action nudge), producing an `overallSignal` (STRONG_BUY … STRONG_SELL) and a **confidence 0–95%** derived from dominant-score strength + agreement + a base floor.
- **`calculateOptionMetrics`** — simplified Greeks/moneyness: delta approximation, intrinsic/time value, ITM/ATM/OTM classification, and a risk-reward estimate.

### `TradingScanner` (`scanner.js`)
- **Intraday options scanner** — `setInterval` (default 2s) that checks daily limits, fetches prices, rotates NIFTY/BANKNIFTY, pulls candle history, runs the comprehensive signal, fetches the option chain, and (if confidence ≥ threshold and no active trade) generates a signal: picks 1-strike-OTM CE/PE, sets entry band, 15% target, 15% stop-loss, and sizes lots from the per-trade risk budget.
- **BTST stock scanner** — scores the universe on volume spike (≤30), RSI 45–75 proximity to 60 (≤25), above-VWAP (≤20), and positive momentum (≤25), plus a strong-setup bonus; returns the top 3 with 2% target / 1.5% SL.
- **BTST options scanner** — per index, prefers 1-strike-ITM options (higher delta, less theta impact) for an overnight 10% target / 10% SL, boosting confidence for high delta and rising OI; returns the top 3.
- **Paper-trade lifecycle** — `takeTrade` recomputes target/SL off the actual entry LTP; `updateActiveTrade` re-prices from the live chain, computes P&L, and flags TARGET_HIT / STOP_LOSS_HIT; `exitTrade` books P&L into `dailyStats`, capital, and history.

### `SilverPredictionEngine` (`server.js`)
Multi-timeframe (5m intraday + 1h swing) engine over **10 indicators** — RSI (75/25 thresholds), fast MACD (8/17/9), Bollinger (2.5σ), ADX (used as a 0.7–1.25× trend-strength multiplier), moving averages (EMA 9/21 + SMA 50/200 golden/death cross), VWAP (intraday), volume-price, candlesticks, pivots, and the **gold/silver ratio** (from `GC=F`). Per-timeframe scores are summed, ADX-multiplied, then combined 60/40 (intraday/swing) with an alignment multiplier into a direction + signal + confidence. Indian ETF predictions blend **70% COMEX score + 20% USD/INR trend + 10% ETF volume-price**, with COMEX-based fallbacks when NSE is closed.

## Security Notes

- **OAuth code exchange** happens entirely server-side (`/callback` and `getUpstoxAccessToken` POST to Upstox's token endpoint); the browser never handles the client secret or the resulting token.
- **Token in memory only** — the access token lives in `UPSTOX_CONFIG.accessToken` with an expiry check; nothing is persisted, so a restart forces re-auth.
- **Credential hygiene (known smell, being remediated).** The Upstox `apiKey`/`apiSecret` and the OAuth `redirectUri` are currently hardcoded in `server.js`, and a helper shell script and local `cert.pem`/`key.pem` exist for the ngrok/HTTPS dev setup. These should be moved to environment variables (e.g. `process.env.UPSTOX_API_KEY`) and the certs/secrets kept out of version control. *(Values are intentionally not reproduced here.)*
- **CORS is currently open** (`app.use(cors())`) for local development; a production deployment should restrict origins.

## Notable Patterns

- **Proxy / Backend-for-Frontend (BFF)** — one backend normalizes three heterogeneous upstreams into a single client contract.
- **Shared module** — environment-guarded `indicators.js` runs identically in Node and the browser.
- **Strategy-style scanners** — intraday / BTST-stock / BTST-option scanners share primitives but encapsulate distinct entry logic and risk parameters.
- **Graceful degradation / fallback ladders** — Upstox → NSE → simulation for chains; NSE → Yahoo → constants for indices; live → simulated for the whole client.
- **Singletons + callback wiring** — each layer exposes one instance; the scanner pushes updates to the UI via `onSignal`/`onUpdate` callbacks rather than a framework.
