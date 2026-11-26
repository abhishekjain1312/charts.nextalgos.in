# charts.nextalgos.in

Charting module for NextAlgos (Indian markets), with candle history from Upstox and Dhan.

## Status (26 Nov 2026)

Design only. Nothing is built yet.

- `docs/specs/charts.md` is the draft spec. Every line in it is tagged **[Verified]**, **[Unverified]** or **[Guess]**.
- The spec refers to code in the main app repo, `abhishekjain1312/Nextalgos.in` (`server/src/...`, commit `dc8ff47`). That repo was not changed.
- The owner's decisions D-C1 to D-C6 are open (spec §10).
- What must be checked before any code is written is in spec §11.

## Where to pick up

1. The owner answers D-C1 to D-C6.
2. Check that the owner's Dhan account has the Data API subscription. Without it, Dhan cannot serve charts.
3. Make one real history request to each broker to settle the [Unverified] candle-anchor and timestamp lines.
4. Then build, in the main repo. Expect three changes:
   - a migration adding `scrip_master.dhan_instrument`
   - a shared `bar-buckets` module
   - a `/api/charts/bars` route, plus the client chart
