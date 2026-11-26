# Charts: candle history from Upstox and Dhan

Status: **Draft for the owner's review, 26 Nov 2026. Nothing built.** No code,
migration, push or pull request until the owner says so.

Every statement below is tagged:

- **[Verified: source]**: read in this repository or in the broker's own published page, on 26 Nov 2026.
- **[Unverified]**: believed true but not read from a primary source here. Must be checked before the line of code that depends on it is written.
- **[Guess]**: a design assumption. The owner decides, or a real reply settles it.

The platform rule applies throughout: real data or an honest empty state, never a
fabricated candle (PRODUCT.md:27).

---

## 1. What exists today

- **No chart history is fetched from any broker.** No call to Upstox's or Dhan's history endpoints exists in `server/src`. **[Verified: grep for `charts/`, `historical`, `historical-candle`]**
- Candles are built only from live ticks: `CandleBuilder` (`server/src/services/marketdata/CandleBuilder.ts`) emits `candle:1m`, `candle:5m`, `candle:15m`, bucketed on the **wall clock** (`Math.floor(ts / ms) * ms`), and `MarketDataHub.ts:1862-1867` broadcasts them to screens. **[Verified]**
- The strategy engine builds 3m, 10m, 30m and 1h itself, **anchored to the session open**, in `server/src/services/algo/engine/bars-cache.ts` (`bucketOf`, `gather`, `take`). The comment there states the reason: the first 30-minute NSE candle is 09:15–09:45 on a chart, and 09:00–09:30 on the clock. **[Verified: bars-cache.ts:8-14, 186-233]**
- Minutes are kept on disk for 45 days (`KEEP_DAYS`, `candle-store.ts:36`), and only for indices, F&O underlyings and running strategies' contracts (owner's D6, `candle-scope.ts`). That store cannot back a chart of an arbitrary instrument. **[Verified]**
- `strategies-client` has no charting library (`package.json`: react, react-dom only). **[Verified]**
- Prices inside the platform are integer paise; brokers' rupees are converted at the boundary (`dhanPaise`, `dhan-api.ts:136`). **[Verified]**

## 2. Dhan instrument naming (what the order code does)

- **Orders do not send an instrument type to Dhan.** `routeDhan` (`OrderManager.ts:2344`) names the instrument by `exchangeSegment` + `securityId` only. Margin (`margin.ts:257`) and convert (`convert.ts:97`) do the same. **[Verified]**
- Segments are Dhan's verbatim: `IDX_I`, `NSE_EQ`, `NSE_FNO`, `NSE_CURRENCY`, `BSE_EQ`, `BSE_FNO`, `BSE_CURRENCY`, `MCX_COMM` (`DhanSegment`, `dhan-api.ts:38-46`), stored per scrip as `dhan_segment` with `dhan_security_id`. **[Verified]**
- Dhan's raw instrument names are read from the master's `INSTRUMENT` column and **collapsed and discarded** at load (`dhan-scripmaster.ts:59-68`): `INDEX→IDX`, `EQUITY→EQ`, `FUTIDX/FUTSTK/FUTCOM→FUT`, `OPTIDX/OPTSTK/OPTFUT→OPT`. Any other value is skipped. `scrip_master.instrument_type` holds only `EQ | FUT | OPT | IDX` (`scrip.repo.ts:18`). **[Verified]**
- **Consequence for charts.** Dhan's history requests require an `instrument` field (see §3.2), which the order path never needed. The raw name is not stored anywhere today. **[Verified, both]**

**Decision proposed (D-C1):** store the raw Dhan name verbatim (the annexure's `instrument` values are the master's `INSTRUMENT` values, character for character) in a new `scrip_master.dhan_instrument` column, written by the Dhan master parser, the same way `upstox_instrument_key` is stored whole and never rebuilt (`upstox-instruments.ts:8-12`). Do **not** derive it from `instrument_type` + exchange: FUT on NFO can be FUTIDX or FUTSTK, and OPT on MCX is OPTFUT, so derivation is a guess per row. One migration, one parser line, no change to the collapsed `instrument_type` the order path and risk rules read.

Side note, not in scope: `BodSelector.ts:80-249` still filters on `'FUTIDX','OPTIDX'` alongside `'FUT','OPT'`. Since the Dhan parser no longer writes those values, those branches look dead. **[Guess: whether another master writes them was not checked]**

## 3. What each broker serves

### 3.1 Upstox (v3)

Source: [Historical Candle Data V3](https://upstox.com/developer/api-documentation/v3/get-historical-candle-data), [Intraday Candle Data V3](https://upstox.com/developer/api-documentation/v3/get-intra-day-candle-data), read 26 Nov 2026.

| | Historical | Intraday (today) |
|---|---|---|
| Call | `GET https://api.upstox.com/v3/historical-candle/{instrument_key}/{unit}/{interval}/{to_date}/{from_date}` **[Verified]** | `GET .../v3/historical-candle/intraday/{instrument_key}/{unit}/{interval}` **[Verified]** |
| Units | `minutes` (1–300), `hours` (1–5), `days`/`weeks`/`months` (1) **[Verified]** | `minutes` (1–300), `hours` (1–5), `days` (1) **[Verified]** |
| Dates | `YYYY-MM-DD`; `to_date` inclusive; `from_date` optional **[Verified]** | none; "the current trading day" **[Verified]** |
| Depth | minutes/hours from Jan 2022; days/weeks/months from Jan 2000 **[Verified]** | today |
| Per request | 1–15 min: 1 month. >15 min: 1 quarter. hours: 1 quarter. days: 1 decade. weeks/months: no limit **[Verified]** | not stated |
| Candle | `[timestamp, open, high, low, close, volume, oi]`, timestamp ISO with `+05:30`, "start time of the candle" **[Verified]** | same **[Verified]** |
| Rate limit | not stated on the page **[Unverified]** | not stated |

- Instrument is `scrip_master.upstox_instrument_key`, already stored. **[Verified]**
- Whether Upstox anchors a 30-minute or 1-hour candle at 09:15 or at 09:00 is **not stated**. **[Unverified: settle with one real request of NIFTY 50, 30 minutes, any past day, and read the first timestamp]**
- Whether the historical call includes today, or only the intraday call does, is not stated verbatim. **[Unverified]** Design below calls both and merges on timestamp, so it is right either way.
- Prices in rupees, converted to paise at the boundary. **[Guess: the page gives sample numbers in rupees; units not stated in words]**

### 3.2 Dhan (v2)

Source: Dhan's own reference, read in a browser on 26 Nov 2026: [Historical Data](https://docs.dhanhq.co/api/v2/historical-data/), [Get Daily Historical Data](https://docs.dhanhq.co/api/v2/historical-data/get-daily-historical), [Get Intraday Historical Data](https://docs.dhanhq.co/api/v2/historical-data/get-intraday-historical), [Annexure](https://docs.dhanhq.co/api/v2/annexure), [Rate Limits](https://docs.dhanhq.co/api/v2/rate-limits).

| | Daily | Intraday |
|---|---|---|
| Call | `POST https://api.dhan.co/v2/charts/historical` **[Verified]** | `POST https://api.dhan.co/v2/charts/intraday` **[Verified]** |
| Body | `securityId` (string), `exchangeSegment`, `instrument`, `expiryCode` (int), `oi` ("true"/"false" string), `fromDate`, `toDate` **[Verified]** | `securityId`, `exchangeSegment`, `instrument`, `interval`, `oi`, `fromDate`, `toDate` **[Verified]** |
| Segments allowed | `NSE_EQ`, `NSE_FNO`, `BSE_EQ`, `BSE_FNO`, `MCX_COMM`, `IDX_I` **[Verified]** | same **[Verified]** |
| Interval | daily | `"1"`, `"5"`, `"15"`, `"25"`, `"60"` minutes (sent as a string in the sample) **[Verified]** |
| Dates | `YYYY-MM-DD`; **toDate non-inclusive** **[Verified]** | `YYYY-MM-DD`. Inclusivity not stated in words; the sample asks 13 to 14 Nov for "13 November", so it reads non-inclusive **[Verified sample; inclusivity Unverified]** |
| Depth | "back upto the date of its inception" **[Verified]** | "last 5 years", "for all active instruments" **[Verified]** |
| Per request | not stated | **90 days** "at once for any of the above time intervals" **[Verified]** |
| Response | parallel arrays `open`, `high`, `low`, `close` (float, rupees), `volume`, `open_interest` (int), `timestamp` (int, **epoch seconds**) **[Verified]** | same **[Verified]** |
| Access | "Data APIs need an active Data API subscription on your account." **[Verified]** Refusal codes: `DH-902` / Data API `806` "Data APIs not subscribed" **[Verified, annexure]** | same |
| Rate limit | Data APIs: **5 req/sec, 100,000 per day**. Over the limit: `errorCode: "RL001"`, "retry after 1 second"; also `DH-904` / `805` **[Verified]** | same bucket **[Guess: the page lists one "Data APIs" row and does not name the history calls in it]** |

- **`instrument` values** (annexure, verbatim): `INDEX`, `FUTIDX`, `OPTIDX`, `EQUITY`, `FUTSTK`, `OPTSTK`, `FUTCOM`, `OPTFUT`. **These are exactly the eight strings the Dhan master's `INSTRUMENT` column carries and the parser reads (`dhan-scripmaster.ts:59-68`).** So D-C1 is a straight copy of that column, no mapping. **[Verified on both sides]**
- **`expiryCode`** (annexure): `1` Near, `2` Next, `3` Far. The sample sends `0` for an equity. For a chart of a specific contract by its own `securityId`, what `expiryCode` changes is not stated. **[Unverified: send `0`, confirm with one real F&O request]**
- **Timestamp = candle start, IST-aligned.** The illustrative intraday sample's first stamp `1794541500` is 03:45 UTC = **09:15 IST** on 13 Nov 2026; the daily sample's `1794249000` is 18:30 UTC = **00:00 IST** on 10 Nov 2026. **[Verified from samples only; the docs say "Epoch timestamp", nothing more. Confirm on the first real reply]**
- Whether Dhan's **60-minute** candle starts at 09:15 (09:15, 10:15, …) or on the clock hour is **not stated**. **[Unverified]** The 5-minute sample starts at 09:15, which says nothing about 60.
- Expired contracts have their own page ("Expired Options Data", linked from the intraday page). Not read; out of v1.
- A sandbox exists at `https://sandbox.dhan.co/v2`. **[Verified]** Useful for parser fixtures without spending the live budget. **[Guess: whether the sandbox returns real candles is not stated]**

## 4. Bar sizes on the chart, and where each comes from

Chart intervals offered: **1m, 3m, 5m, 10m, 15m, 30m, 1h, 1D, 1W, 1M**. The intraday set is the engine's (`INTERVALS`, `ir/catalog.ts:221`) so a chart and a strategy read the same candles. **[Guess on 1D/1W/1M: owner to confirm the list]**

| Interval | Upstox | Dhan |
|---|---|---|
| 1m | native `minutes/1` | native `1` |
| 3m | native `minutes/3` (anchor to check) | **built from 1m** |
| 5m | native | native `5` |
| 10m | native `minutes/10` (anchor to check) | **built from 1m** (or 5m) |
| 15m | native | native `15` |
| 30m | native `minutes/30` (anchor to check) | **built from 15m** |
| 1h | native `hours/1` (anchor to check) | **built from 15m**; native `60` only if it is shown to start at 09:15 |
| 1D | native `days/1` | native daily |
| 1W | native `weeks/1` | **built from daily** |
| 1M | native `months/1` | **built from daily** |

Rule: **a candle begins where a trader's chart says it begins: the session open**, as the engine already does. If Upstox's native 30m/1h turn out clock-anchored, Upstox builds them from 15m too. Building from 15m is safe for 30m and 1h because the NSE/BSE/NFO/BFO open (09:15) and the MCX open (09:00) both sit on a 15-minute boundary (`market-hours.ts:43-54`). **[Verified for the sessions as coded; Guess that each broker's 15m is itself session-anchored, which holds automatically since both opens are on a 15-minute boundary of the clock]**

25-minute candles from Dhan are not offered: nothing else on the platform uses them. **[Guess]**

### 4.1 Helpers to reuse

From `server/src/utils/market-hours.ts`, all exported and tested by the engine today **[Verified]**:

- `segmentOf(exchange)`: exchange string to `NSE | BSE | NFO | BFO | MCX`, unknown falls to MCX's widest window.
- `sessionOpenMinutes(segment)`, `sessionCloseMinutes(segment)`: minutes past midnight IST. NSE/BSE 09:15–15:30, NFO/BFO 09:15–15:40, MCX 09:00–23:55.
- `istMinutesOf(instant)`, `istDay(instant)`: IST minute of day and IST date.
- `previousWeekday(day)`: for walking back across weekends when paging.

From `server/src/services/algo/engine/bars-cache.ts`, **private today** **[Verified]**:

- `bucketOf(ts, minutes, opens, closes)`: the session-anchored bucket a bar falls in; the day's last bucket is cut at the close (an hourly NSE candle from 15:15 is 15 minutes long).
- `gather(bars, start, end)`: one OHLCV bar from the bars inside a bucket; OI is the latest non-null.

**Proposed refactor (needs the owner's yes, it touches engine code):** move `bucketOf` and `gather` unchanged into a new pure module `server/src/services/marketdata/bar-buckets.ts`, export them, and have `bars-cache.ts` import them. The engine's behaviour does not change; its existing candle tests (`algo-candle-lengths.test.ts`, `candle-close.test.ts`) must stay green. Charts then call one function:

```ts
// Pure. Input oldest first, all bars of one instrument at one source length.
rollUp(bars: Bar[], targetMinutes: number, exchange: string): Bar[]
```

Weekly and monthly from daily need a new small helper: week starts Monday IST, month by IST calendar month, candle `ts` = first trading day present. **[Guess on week anchor: Monday is the common chart convention; owner to confirm]**

Known edges of reusing these helpers, carried knowingly:

- **No holiday calendar** (`market-hours.ts:19-22`). For charts this costs nothing: buckets are built only from bars that exist.
- **MCX close is the widest envelope (23:55)**, not the DST-dependent 23:30. The last hourly MCX candle on a 23:30 day is 23:00–23:30, correct either way, since no bars exist after 23:30.
- **NFO/BFO close is 15:40** (closing auction since 3 Aug 2026), NSE cash 15:30. An index (`exchange = NSE`) and its future (`NFO`) cut their last candle at different times. That matches the exchanges. **[Verified as coded]**
- **Special sessions** (muhurat, Saturday budget session) fall outside the open/close assumptions. A muhurat hour in the evening would bucket off a 09:15 anchor. **[Guess: acceptable for v1, flagged on the chart only if it happens]**

## 5. Live tail

A chart's history ends at "now"; the forming candle comes from the live feed.

- The tick feed already reaches the screens. The chart subscribes to the instrument's ticks and updates the last bar in the browser using the **same bucket rule** as the server (one shared bucket function, ported to the client or the bucket start sent by the server). **[Guess on where the bucket is computed]**
- `candle:5m` / `candle:15m` from `CandleBuilder` are clock-bucketed; this coincides with session buckets only because both opens sit on 5- and 15-minute clock boundaries. They are **not** used for 3m (09:15 is a 3-minute boundary too, but nothing guarantees it for future sessions), 10m, 30m or 1h. **[Verified on the arithmetic]**
- At each closed minute the chart re-checks against history once per few minutes (or on reconnect), so a tick lost in transit does not leave a wrong bar standing. **[Guess on cadence]**

## 6. Which broker answers

Both need a logged-in account (Upstox token; Dhan client id + token). **Dhan also needs the paid Data API subscription** on that account (§3.2). **[Verified]** Whether Upstox's history needs anything beyond the login is not stated on its pages. **[Unverified]**

Proposed order (D-C2, owner decides):

1. **Upstox first** when the owner has a connected Upstox account: every interval native or from 15m, minutes back to Jan 2022, daily to Jan 2000, no extra data subscription named in its docs.
2. **Dhan** when there is no Upstox account, or Upstox fails for that instrument.
3. Neither: honest empty state naming which account to connect.

Note: Dhan's 90 days per 1m call (verified) is longer than Upstox's 1 month, so for deep 1-minute history Dhan needs fewer calls. Upstox still goes first for coverage of intervals. **[Guess: owner may prefer the reverse]**

A chart never splices two brokers' candles into one series. The source is shown on the chart ("Data: Upstox"). Different vendors' volume and OI can differ for the same minute. **[Guess on the rule; strongly recommended]**

## 7. Server API

`GET /api/charts/bars?scrip=<scripId>&interval=<1m..1M>&to=<ISO>&limit=<n>`

- Returns `{ source: 'upstox' | 'dhan', interval, bars: Bar[] (oldest first, paise, ts = candle start epoch ms), complete: boolean, olderAvailable: boolean }`.
- Paging backwards: the client asks again with `to` = oldest `ts` it holds.
- The server maps `interval` to the broker request (§4), splits the date range by the broker's per-request limit, calls sequentially within a per-broker rate budget, converts rupees to paise, rolls up where §4 says "built", and drops the forming candle unless `to` is now.
- Errors are the broker's words: "Upstox: instrument not found", "Dhan: Data API not subscribed on this account (806)", never an empty array presented as "no trades".
- **Expired contracts** are out of v1. Dhan's intraday call covers "all active instruments" and expired ones have a separate "Expired Options Data" page (not read). Upstox's expired-contract route was not read. **[Verified for Dhan's scope sentence; the rest Unverified]**

Cache (D-C3, owner decides): an in-memory LRU per (source, scrip, source interval, day), with closed past days kept and today's re-fetched. No new table in v1. A disk table later if the request budget proves tight. **[Guess]**

## 8. Client

- Charting library: **lightweight-charts** (TradingView's open-source library) is the obvious fit for candles + volume + live update with no backend. Its licence and attribution terms must be read before adding it. **[Unverified: licence terms]** Owner may prefer another.
- Chart page/panel: instrument search (existing scrip search), interval switcher, candles + volume, OI pane for F&O, crosshair OHLCV readout in rupees, source badge, scroll-left loads older history.
- Empty and error states follow PRODUCT.md:27: each says what is missing and why.

## 9. Tests (when built)

- `rollUp`: 3m/10m from 1m, 30m/1h from 15m, last candle cut at 15:30 (NSE), 15:40 (NFO), MCX evening; a missing minute inside a bucket; OI is the bucket's last non-null.
- Weekly/monthly from daily across a holiday Monday and a month that starts on a weekend.
- Upstox and Dhan response parsers against **real recorded replies** (one per broker, captured once, stored as fixtures), never invented ones.
- Engine candle tests stay green after `bucketOf`/`gather` move.

## 10. Decisions for the owner

| # | Question | Recommendation |
|---|---|---|
| D-C1 | Store Dhan's raw instrument name in a new `dhan_instrument` column | Yes |
| D-C2 | Which broker first | Upstox, then Dhan, never mixed in one series |
| D-C3 | Cache in memory only for v1 | Yes |
| D-C4 | Move `bucketOf`/`gather` out of `bars-cache.ts` into a shared module | Yes, unchanged |
| D-C5 | Interval list incl. 1D/1W/1M; week anchored Monday | Yes |
| D-C6 | Charting library | lightweight-charts, after its licence is read |

## 11. Must be verified before building (no code depends on a guess)

1. Dhan, one real reply each: timestamps are candle starts in IST as the samples show; intraday `toDate` is non-inclusive; `expiryCode: 0` is right for a specific F&O contract; whether the 60-minute candle starts at 09:15.
2. Whether the owner's Dhan account(s) carry the Data API subscription. Without it, Dhan cannot be a chart source at all.
3. Upstox, one real reply: whether native 3m/10m/30m/1h candles start at 09:15.
4. Upstox's rate limit for history calls (not on the pages read).
5. lightweight-charts licence terms.
