# Tennis in-play pricing and API research

Reviewed September 28, 2026. Research, documentation and synthetic calculations only. No bets, purchases, new continuous collectors or model training were started. Current ATP/WTA market availability was checked through small read-only metadata probes; no quote archive was collected. The existing cloud-only storage requirement remains in force.

## What a swing means

The useful question is whether an executable price is wrong given information we actually possess. A large move does not establish that it will reverse. Tennis converts points into games, sets and matches nonlinearly; score, server, both players' serving ability, match format and tiebreak rules all matter. A pre-match favorite can correctly become an underdog after losing a set.

For an illustrative standard advantage game, assume the server independently wins each point with probability 0.65. These are our synthetic calculations, not observed player estimates or match-winning probabilities:

| Server's current score | Probability of holding this game |
| --- | ---: |
| 0–0 | 82.96% |
| 0–15 | 68.92% |
| 0–30 | 47.66% |
| 0–40 | 21.29% |
| 15–40 | 32.75% |
| 30–40 | 50.39% |
| Deuce | 77.52% |

At 30–40, winning the next point raises the hold probability to 77.52%; losing ends the game. The current 50.39% is the probability-weighted average of those outcomes. Expected price recovery after a successful point is already paid for by the risk of losing it.

The calculation uses `D = p² / (p² + (1-p)²)` at deuce and `H(a,b) = p H(a+1,b) + (1-p) H(a,b+1)` elsewhere, with game-win/loss absorbing states. The independent-point assumption is illustrative. A production match model must also handle serve rotation, first/second serve where observable, best-of-three/five, no-ad formats, final-set rules, and retirement contingencies.

The [Klaassen–Magnus score model](https://www.sciencedirect.com/science/article/pii/S0377221702006823) is a useful foundation. Their [point-dependence research](https://www.janmagnus.nl/papers/JRM057.pdf) finds deviations from strict independence while supporting its use as an approximation. [Dynamic serve-strength research](https://www.sciencedirect.com/science/article/pii/S0169207017301395) motivates gradually updating pre-match ability estimates, with shrinkage instead of treating two bad points as a new ability level.

Evidence does not establish a generic fade-the-break strategy. An [older Australian Open study](https://www.sciencedirect.com/science/article/pii/S0169207009001721) found generally score-consistent pricing and some under-adjustment following breaks. A [tennis fast-trading study](https://www.sciencedirect.com/science/article/pii/S0167268118302683) found that delayed entries could erase apparent opportunities. These studies concern other periods/markets, not a verified current Kalshi edge.

## Entry hypotheses to test

Priority is an engineering/research judgment, not a finding of profitability. Each setup requires a fresh score, a valid book, an exact contract mapping, costs and uncertainty.

| Setup | Required signal | Why it might fail |
| --- | --- | --- |
| Changeover or set break | Score-adjusted value remains above executable ask through a stable observation window | Stable score does not imply an uninformed market or fresh feed |
| Favorite broken early | Recomputed match probability exceeds the new price after the break and updated serve evidence | The break or a real deterioration may justify the entire drop |
| Close first-set loss | Format-aware match model shows residual mispricing after incorporating the lost set | Pre-match favoritism alone is insufficient; fewer sets remain |
| Server behind 0–15/0–30 | Updated value exceeds the price of the exact game/match contract | Confusing high initial hold probability with current hold probability |
| Tiebreak service-order discrepancy | Correct next server and target score produce a persistent price discrepancy | Wrong score, service sequence, format or delayed observations |
| Persistent serve deterioration | Shrunk live evidence changes estimated serve strength beyond model uncertainty | Noise, selection and unobserved injury can dominate |
| Price move without confirmed score change | Both feeds are fresh and the move exceeds model/cost uncertainty | The market may know a point or other information before our feed |
| First-serve fault | Fault event is received before second serve, and updated point/match value exceeds costs | A field attached to the completed point is too late; this is a demanding latency race |

Start with stable-state and completed-game observations. First-serve entry remains an acquisition experiment until pre-second-serve delivery is demonstrated. Neither an unchanged scoreboard nor a television picture establishes that no point has occurred.

For a binary $1 outcome held to settlement, estimated profit per contract is `p(win) - executable ask - expected fees`. Use `E[settlement payout]` instead of `p(win)` when fair-price/partial settlements are possible. Buying at 48 cents with a hypothetical calibrated 54% win estimate leaves six cents before costs and model error; those figures are illustrative, not a quote or recommendation.

For an intended short trade, the target is `E[executable exit bid] - entry ask - entry fees - exit fees`. A model of eventual victory does not by itself model the future exit bid. Do not subtract spread twice if entry ask and exit bid already incorporate it. Use the depth-weighted execution price for size, and account for delays, missed fills and uncertainty separately.

## Kalshi: a verified technical route

Small metadata requests on September 28 at 08:23:58 UTC (ATP) and 08:24:10 UTC (WTA) returned one open/active match-winner contract in each series, with further-page cursors. This establishes available markets, not complete tour coverage, in-progress match status, liquidity or executable prices. See [ATP series](https://external-api.kalshi.com/trade-api/v2/series/KXATPMATCH) and [WTA series](https://external-api.kalshi.com/trade-api/v2/series/KXWTAMATCH).

The [recommended environments](https://docs.kalshi.com/getting_started/api_environments) are:

```text
REST: https://external-api.kalshi.com/trade-api/v2
WebSocket: wss://external-api-ws.kalshi.com/trade-api/ws/v2
```

Requests below describe a future permitted cloud integration; no local collector is being instructed or enabled:

| Need | Route |
| --- | --- |
| ATP discovery | GET /markets?series_ticker=KXATPMATCH&status=open&limit=100; follow cursor |
| WTA discovery | GET /markets?series_ticker=KXWTAMATCH&status=open&limit=100; follow cursor |
| Rules and identity | GET /markets/{ticker}, GET /series/{series_ticker}, corresponding event metadata |
| Depth snapshot | GET /markets/{ticker}/orderbook |
| Executed trades | GET /markets/trades?ticker=... with documented time/cursor filters |
| Continuous updates | WebSocket orderbook_delta, trade and appropriate lifecycle/status subscriptions |
| Older records | GET /historical/cutoff, then type-appropriate historical endpoints |

Some [REST market data](https://docs.kalshi.com/getting_started/quick_start_market_data) needs no key. [WebSocket connections](https://docs.kalshi.com/websockets/websocket-connection) require signed authentication, including public-data channels. [Key documentation](https://docs.kalshi.com/getting_started/api_keys) describes Ed25519 and RSA-PSS support; SDK compatibility must be checked separately. Credentials belong in cloud secrets, not chat, source files or logs. The [demo environment](https://docs.kalshi.com/getting_started/demo_env) is for integration testing and does not prove real-market liquidity or fills.

The [REST book](https://docs.kalshi.com/getting_started/orderbook_responses) contains YES and NO bids. In dollars, YES ask = `1 - best NO bid`; NO ask = `1 - best YES bid`. Preserve fixed-point price/quantity strings with decimal arithmetic. Missing opposing bids mean unknown/unavailable asks, not free liquidity. [Book updates](https://docs.kalshi.com/websockets/orderbook-updates) require snapshot initialization and sequence tracking; invalidate state after a gap until recovered. A historical last trade or candle is not an executable offer.

Keep [trade IDs and execution fields](https://docs.kalshi.com/api-reference/market/get-trades) distinct from quote updates. Contract volume is not unique bettors or necessarily informed directional demand. Separate block transactions where appropriate. [One-minute candles and historical routing](https://docs.kalshi.com/getting_started/historical_data) cannot reconstruct post-serve depth or queue priority. No historical full-depth endpoint was established in this review.

[Current rate limits](https://docs.kalshi.com/getting_started/rate_limits) use token budgets, not a universal requests-per-second figure. Basic authenticated limits are documented as 200 read/100 write tokens per second, with typical cost 10 tokens per request and endpoint exceptions. Use actual account limits, bounded exponential backoff and reconnect recovery.

## Fees, settlement and data rights

The [fee schedule effective July 7, 2026](https://kalshi.com/docs/kalshi-fee-schedule.pdf) includes maker fees for ATP/WTA match series. With multiplier one, its formulas before rounding are `0.07 × contracts × price × (1-price)` for taker fills and `0.0175 × contracts × price × (1-price)` for applicable maker fills. At 50 cents, this is approximately 1.75 cents versus 0.4375 cents per contract before rounding. A taker round trip near that price costs approximately 3.5 cents in fees alone. Check [event overrides](https://docs.kalshi.com/api-reference/events/get-event-fee-changes), effective rules and actual charged fees; resting orders are not automatically free, and touching a limit does not prove a fill.

The two reviewed match-winner examples distinguish cancellation before any ball is played from withdrawal after play starts. Their current series link the generic [ACHIEVEMENTS contract](https://assets.kalshi.com/contract_terms/ACHIEVEMENTS.pdf). Never select a rule document by filename alone. Match, set, total and spread contracts can have different retirement treatment; undetermined derivative outcomes can resolve at fair price. Preserve exact rules/version and official settlement value. Do not label every retirement a refund or 50-cent payout. [Example total-games market](https://kalshi.com/markets/kxwtagtotal/wta-total-games/kxwtagtotal-26sep23yexbir).

The [Developer Agreement](https://kalshi-public-docs.s3.amazonaws.com/Kalshi-Developer-Agreement.pdf) limits API use/storage to a member's own Kalshi trading and prohibits sharing without written authorization. Separate [Data Terms](https://kalshi-public-docs.s3.amazonaws.com/kalshi-data-terms-of-service.pdf) contain broad ML/AI restrictions. No reviewed text establishes a model-training exemption or clear supersession. Technical access therefore does not authorize this project's model/public-archive use. Keep Kalshi raw observations out of public GitHub releases; obtain written clarification for the intended modeling and private retention scope before enabling that pipeline. No third-party mirror is assumed to remove these restrictions.

## The second feed: tennis state

### Zero-additional-cost option, reviewed September 28

The owner prefers a free route. A limited prototype can use synthetic scoring calculations, occasional live-score snapshots, permitted public quote access and paper decisions. No paid provider, funded trading account, GPU or new paid cloud resource is needed for that scope. This does not establish a free complete feed suitable for racing the next serve.

[Live Tennis API](https://livetennisapi.com/subscribe/free), a different provider from API-Tennis below, advertises a permanent free tier with no card: 100 requests/day and 30/minute. Its [live-data documentation](https://livetennisapi.com/tennis-live-data-api) includes sets, games, points, server and tiebreak state. Completed history, point-event records and streaming are paid capabilities. No account was created and actual coverage/latency has not been tested. All fields must be checked against the reference schema rather than inferred from names.

At one page per refresh, polling every 30 seconds uses 100 calls in 50 minutes. Across two hours, the same allowance permits roughly one call every 72 seconds before discovery, pagination and retries. Use fewer scheduled observations and manual score confirmation for an initial experiment; missing points remain missing. Do not rotate accounts to exceed the cap.

Its [terms](https://livetennisapi.com/terms) permit use within applications but prohibit raw bulk republication and limit retained data to operational need. Free access is not permission for a public CSV archive or indefinite model-training history. Confirm retention/model scope before collecting a research archive; keep restricted observations out of public releases.

[Kalshi public REST](https://docs.kalshi.com/getting_started/quick_start_market_data) requires no key for documented market/book reads. Its model-use and sharing restrictions above still apply. The [Kalshi demo](https://docs.kalshi.com/getting_started/demo_env) supplies mock funds for API tests; demo prices/fills are not evidence of a real-market edge. Public Polymarket price streams are another access lead, subject to their terms; their sports stream is not a verified complete tennis point feed.

[Standard GitHub-hosted runners in public repositories](https://docs.github.com/en/billing/concepts/product-billing/github-actions) have free execution minutes. Storage, larger runners and private-workflow quotas are separate. Keep jobs finite, avoid paid runners/GPU, and verify storage/quota and zero-spend controls before dispatch. Public compute must not expose restricted data through logs, artifacts or releases. Nothing in this addendum deploys a collector or changes billing settings.

### More detailed paid feeds, if later justified

Kalshi's documented [game-stats coverage](https://docs.kalshi.com/api-reference/live-data/get-game-stats) does not include tennis. Generic live-data endpoints do not establish point-level tennis/server coverage. We need an independent permitted feed.

| Candidate | What is documented | Practical limit |
| --- | --- | --- |
| [API-Tennis REST](https://api-tennis.com/documentation) and [WebSocket](https://api-tennis.com/documentation_websocket) | Live score, server, sets and point sequence; get_fixtures/get_livescore and wss://wss.api-tennis.com/live | Published samples lack a separate point occurrence timestamp and reliable first/second-serve state. Match-level event_time is not a point clock. Stamp receipt time ourselves. |
| [Sportradar live timeline](https://developer.sportradar.com/tennis/reference/live-timelines) | Event IDs/times, score, server, first_serve_fault and interruption/status information | Coverage/access are tier-dependent. One-second cache TTL is not a latency guarantee, nor proof a fault arrives before the second serve. |

API-Tennis currently lists [Business at $80/month](https://api-tennis.com/) with WebSockets and advertises a 14-day trial; confirm trial entitlements, coverage and intended-use rights before signup/purchase. Its [terms](https://api-tennis.com/terms-of-use) do not grant an open redistribution license. It is a candidate for a forward feed test, not a verified complete pre-2025 archive.

Sportradar [push access](https://developer.sportradar.com/tennis/docs/tennis-ig-push) requires appropriate realtime access, with sales-arranged evaluation. Its [terms](https://developer.sportradar.com/sportradar-updates/page/terms-and-conditions) and contract must cover modeling, betting, retention and distribution. Historical discovery is not a guarantee of full historical coverage. Neither provider has been subscribed to or connected.

Polymarket's [market stream](https://docs.polymarket.com/market-data/realtime-data) is a potential separately reviewed quote source. Its sports stream warns about delayed/missing information and does not establish the complete tennis point clock we need. Global and US products, availability and settlement rules remain separate. Betfair has [paid/restricted live access](https://support.developer.betfair.com/hc/en-us/articles/115003864531-Are-there-any-costs-associated-with-API-access); its delayed key is not a virtual-money sandbox or a suitable latency benchmark.

## Concrete recorder and evaluation specification

Once appropriate access and rights exist, run a finite cloud-only pilot. For restricted feeds, use private cloud storage with agreed retention, not this repository's public release path. No new paid resource or recorder has been created. Keep public licenses and restricted feeds in separate partitions.

Use separate compressed CSV tables rather than duplicating every quote against every point:

```text
match_map.csv.gz:
  canonical_match_id, provider, provider_match_id, player_a_id, player_b_id,
  tour, tournament_id, format, surface, market_ticker, outcome_player_id,
  rule_version_hash, mapping_status

score_events.csv.gz:
  provider_match_id, source_event_id, revision, source_event_time_utc,
  received_at_utc, receipt_sequence, sets_a, sets_b, games_a, games_b,
  points_a, points_b, server_id, tiebreak_target, first_serve_fault,
  match_status, gap_flag

book_events.csv.gz:
  market_ticker, exchange_event_time_utc, received_at_utc, sequence,
  message_type, outcome_side, price_dollars, size, market_status, gap_flag

trades.csv.gz:
  market_ticker, trade_id, exchange_event_time_utc, received_at_utc,
  price_dollars, contracts, taker_direction, is_block_trade

paper_decisions.csv.gz:
  decision_id, received_at_utc, canonical_match_id, market_ticker,
  state_version, model_version, predicted_payout, uncertainty_margin,
  executable_bid, executable_ask, available_size, fee_version,
  decision_reason, hypothetical_order_time, fill_assumption, later_outcome
```

Use stable source IDs, explicit UTC, string IDs, decimal prices and the repository's declared null convention. Sort each stream by receipt time plus deterministic tie-breaker; preserve source event time separately. Missing timestamps stay missing. Treat revisions as new received information, not retroactive changes to the past. Any normalized direction must retain its source-field mapping and API version.

At each simulated decision, join only data received by that time. Apply computation/network delay and applicable exchange processing delay before evaluating executable depth. Never align a point to whichever price move makes a strategy look profitable. A first-serve-fault flag discovered in a completed-point message is unusable for an earlier decision.

First deliver coverage/quality metrics: mapped matches, lost/revised points, reconnect gaps, book validity, quote age and timestamp availability. Court-to-client latency is unknown without an independent clock; do not call receipt-minus-scheduled-start a latency measurement. Test a declared range of additional reaction delays as scenarios, not measurements.

Fit sport abilities on pre-2024 records with suitable rights, calibrate sport probabilities on 2024, then freeze for 2025+ evaluation. This does not establish 2024 Kalshi strategy calibration: no synchronized 2024 Kalshi tennis quote/point archive has been verified. Preregister entry rules before examining evaluation quotes, or explicitly define a new prospective development/test split. Keep whole matches together and account for dependence when reporting uncertainty. New 2026 prospective observations can test a frozen strategy; using them to invent another strategy makes them development data for that new experiment. Report calibration/log loss, executable net returns, drawdown, turnover, fill sensitivity and uncertainty by tour/surface/liquidity. No return estimate is justified by the research in this document.

For Kalshi permission clarification, the concrete scope to request is: use of market quotes alongside an independently trained tennis model for the member's own paper/live trading, private cloud retention and replay, intended retention duration, derived private signals, and whether any aggregate reports can be shared. Ask separately whether training on Kalshi data is allowed. This is a draft scope only; no message has been sent.
