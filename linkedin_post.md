# LinkedIn post (copy below the line)

---

Three months ago I started backtesting short-premium strategies on Indian index options.
The first version made ₹9L+ with a Sharpe of 11.9.

I now know that number was a lie — and finding out why became the actual project.

Over 12 weeks I built a 5-year, 1-minute options store (21 NSE underlyings, ~129k
contracts) and a fixed-strike backtest engine, then spent most of my time killing my own
results. Five separate inflation bugs died on the way:

→ Rolling-ATM marking: valuing a held short against each bar's *current* ATM. Inflated
P&L 2.8×, understated max drawdown 18×.
→ Pre-open stale ticks manufacturing overnight gaps as "profit."
→ A look-ahead fill (bar-open fill on a bar-close signal): 2× inflation, win rate 67% → 47%.
→ Stale prices on a thin leg: an apparently superior structure (Sharpe 3.1 vs 2.3
baseline) had a 97.6% win rate on exactly the trades with carried marks. Clean-priced
trades lost. Retracted.
→ Vendor mark-price smoothing showing theta that traded prices never delivered.

What survived all of it — plus intrinsic settlement of every expiry exit (−9% haircut),
date-accurate STT/txn/GST/brokerage, per-leg slippage stress, and an anchored walk-forward
with train-only selection:

A NIFTY trend book + BANKNIFTY mean-reversion overlay, daily-P&L correlation −0.08.

📊 Sharpe 3.43 full-sample / 2.97 out-of-sample (fixed Sep-2025 split, no peeking)
📊 Calmar 13.9 · 6/6 years positive · 73% positive months
📊 Worst month −12% of deployed capital · ~75–105% annualized on margin
📊 Walk-forward: 5–6/6 folds positive per leg, with *stable* config selection — the
higher-scoring dynamic re-selection variant was rejected for unstable picks (that
instability is the overfit tell, not the total)

The part quants will appreciate: every added layer of realism monotonically shrank every
edge. The strategies that reached zero got documented as null results with the same rigor
as the survivors. The blend is now running as a real-time algorithmic paper trade against
live broker WebSocket feeds — because the one thing no backtest can measure is fill quality.

Signals, parameters, and strikes stay private. The methodology doesn't:
https://github.com/PavanTeja2005/systematic-options-research

Biggest lesson of the quarter: a backtest is guilty until proven innocent. If your Sharpe
is double digits, you haven't found alpha — you've found a bug.

#QuantitativeFinance #AlgorithmicTrading #Options #Backtesting #QuantResearch #NIFTY

---

**Attach (in this order):**
1. `figures/metrics_card.png`   — the stat tiles (thumbnail that stops the scroll)
2. `figures/equity_curves.png`  — combined vs component equity, OOS marker
3. `figures/drawdown.png`       — drawdown profile
4. `figures/monthly_pnl.png`    — monthly distribution
