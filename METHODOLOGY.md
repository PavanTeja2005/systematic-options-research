# Methodology — how the numbers earned trust

The core belief of this program: **a backtest is guilty until proven innocent.** Every
headline number survived a pipeline built to kill it. This file documents the pipeline
and — more importantly — the five bugs and the retractions it produced, because the
graveyard is the credential.

## 1. Data engineering before strategy

- Source: 1-minute rolling-moneyness option series (ATM±10) for 21 NSE underlyings
  (indices + stocks), 5 years, plus real intraday spot OHLCV for all underlyings and
  India VIX (522k 1-min bars).
- The rolling series re-labels strikes intraday, so **highs/lows are unusable** once
  sliced by absolute strike; only closes are point-in-time prints. The store therefore
  ingests **close-only**, keyed by absolute contract identity `(underlying, expiry,
  strike, type)` — ATM is a derived view, never a key.
- Downloaders are resumable and network-poisoning-safe: a chunk is marked complete only
  on a verified 200; the run aborts after consecutive failures rather than recording
  false progress. (This fix was earned: an earlier version silently marked 1,685 failed
  chunks "done" on one underlying and truncated another to 3 of 5 years while reporting
  it complete.)

## 2. The bug graveyard (each one inflated results until found)

| # | Bug | Effect | How it was caught |
|---|---|---|---|
| 1 | **Rolling-strike marking** — held positions valued against each bar's *current* ATM strike instead of the strike actually sold | P&L inflated ~2.8×, drawdowns understated ~18× | A verification script diffing exit prices trade-by-trade against fixed-strike tracking |
| 2 | **Pre-open stale ticks** — stale prior-session prints bucketed into a phantom pre-open bar, manufacturing the overnight gap as profit | One month reported +₹20.4k vs honest +₹1.2k | Scanning the entire store for bars before 09:15 IST |
| 3 | **Look-ahead fills** (an early engine variant) — orders filled at the bar-open timestamp on a signal computed at that bar's close | ~2× inflation; win rate 67% → 47% when fixed | Timestamp audit of fill time vs signal availability |
| 4 | **Sparse-leg stale pricing** — a multi-expiry structure exited on carried (stale) prices for its thinly-traded leg; 97.6% "win rate" on exactly those trades | An apparently superior structure (Sharpe 3.1 vs baseline 2.3), fully **retracted** | Splitting trades by whether the exit price was a real print — the clean-priced trades *lost* |
| 5 | **Vendor mark-price smoothing** — mid/mark candles showed clean premium decay that real traded prices never delivered | An entire candidate strategy line killed: premium collected ≈ premium paid back on buybacks | Re-downloading real traded OHLC from a second source and decomposing the gap |

Two structural lessons generalized: (a) *every* exit marked at a carried price must be
re-settled at intrinsic value before a result is believed (applied as a standard −9 to
−13% haircut to all headline numbers here); (b) each added layer of realism
monotonically shrank every candidate edge — the survivors of all layers are the
portfolio.

## 3. Cost & settlement realism

- Date-accurate Indian F&O charges: STT regime changes (2023, 2024), the Oct-2024
  exchange txn-rate change, SEBI fee, stamp duty, GST, flat ₹20/order brokerage —
  assigned to the correct legs (STT on sells, stamp on buys).
- Real NIFTY lot-size history (75 → 50 → 75 → 65) applied by trade date.
- Both cost bases (with/without brokerage) computed for every experiment; several
  configuration rankings *invert* between the two, and the choice is stated explicitly
  wherever it matters.
- Expiry exits settled at intrinsic value from settlement-day spot, never at last marks.
- Margin quoted from a broker-calibrated model (spread: max-loss + ~2% ELM; verified
  against a live broker quote), not hand-waved.

## 4. Selection hygiene

- **Fixed out-of-sample split** (Sep 2025) used for evaluation only, never selection.
- **Anchored walk-forward** (expanding train window, six 6-month test folds, selection
  on train only) as the final arbiter. The acceptance criterion is not just fold P&L
  but **selection stability** — a real edge keeps choosing the same config as the
  window grows; a re-fit chooses a different one every fold. A dynamic best-pair
  re-selection scheme scored the highest total (₹463k vs ₹252k) and was **rejected**:
  4/6 folds with wandering picks vs 5/6 with a stable pick.
- Documented overfit traps: an India-VIX regime gate whose aggressive "trade extremes
  only" variant was in-sample optimal (Sharpe → 2.5) and collapsed out-of-sample
  (→ 0.4); the gentle variant sitting on a parameter *plateau* was kept. Stops were
  accepted only after confirming a plateau (stable drawdown across a band of stop
  levels), never a single lucky point.
- Single-split OOS is treated as a *screen*, never a verdict — an earlier round of
  single-split "survivors" at Sharpe 8–10 was mostly window luck, exposed by
  years-positive analysis and walk-forward (most failed; the survivors became the book).

## 5. Portfolio construction

- Legs combined by **daily-P&L correlation**, not by adding best performers: the two
  strongest single legs were 0.87 correlated (stacking = leverage, not diversification);
  the final NIFTY-trend × BANKNIFTY-MR pairing runs at **−0.08**.
- Sizing by inverse volatility, then deliberately under-weighted toward the leg with
  the weaker recent regime — the chosen ratio must look right under both the
  full-sample and the recent-window reading.
- Absolute drawdown is treated honestly: diversification buys more return per unit of
  pain, not a smaller worst-case gap. Tail scenarios (an overnight gap hitting all legs
  simultaneously) are outside what historical correlation can promise, and are stated
  as such rather than assumed away.

## 6. Live-parity engineering

Backtests only matter if the live system reproduces them. The (private) execution stack
carries: bit-exact signal-parity tests against the research code (zero mismatches over
311k bars), entry-decision parity (125/125 decisions identical to the backtest),
restart reconciliation against the exchange, fill-confirmation before a position is
committed to state, and a paper → live promotion path. Paper trading on live broker
quotes is the final validation layer — its purpose is to measure the one thing no
backtest can: actual fill quality against the slippage assumptions above.
