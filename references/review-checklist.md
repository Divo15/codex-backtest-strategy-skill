# Strategy compatibility review

Check these points against the generated source and the actual engine before
calling a script compatible.

## Execution boundary

- `STRATEGY_CONTRACT_VERSION` is exactly `"2"`.
- `RUN_MODE` is exactly `"single"` or `"sweep"`.
- `run_strategy(context)` is callable.
- Importing the module has no filesystem, data-loading, network, installation,
  prompt, backtest, or output side effects.
- The script uses `context.market_data`, `context.config`, and
  `context.run_id`; it embeds no developer-specific market-data path.
- Errors propagate instead of becoming partial or invented results.

## Strategy preservation

- Every requested entry, exit, stop, target, trailing,entry-timing, re-entry, sizing,
  expiry, hedge, and timing rule remains implemented.
- Defaults are visible and user-supplied configuration is not silently
  replaced.
- Required data that is unavailable causes a clear failure or documented
  no-trade outcome.
- The fill model, slippage, fees, multiplier, and capital assumptions are
  explicit.

## Completed trades

- Results come from the engine's authoritative closed-trade or closed-leg
  collection.
- The returned count was captured from the same collection and matches exactly.
- Every multi-leg structure returns every leg with a shared `batch_id`.
- Trade IDs are non-empty and unique.
- Sides, actual fills, quantities, multipliers, fees, and timezone-aware times
  map correctly.
- Rows contain no P&L, drawdown, return, win-rate, Sharpe, or other analytics.

## Intraday equity

- Snapshots come from actual engine valuation events, not interpolated or
  reconstructed trade outcomes.
- The first snapshot is zero and precedes the first entry.
- Timestamps are timezone-aware and strictly increasing.
- The final snapshot has zero unrealised P&L and reconciles with closed trades.
- Sampling frequency and any missing-price behavior are disclosed.

## Sweeps

- `SWEEP_PARAMETER_SETS` contains 1–500 unique JSON-serializable mappings.
- All requested combinations across the parameter value lists are present.
- Every key changes or validates a real strategy setting.
- The strategy reads the current mapping from
  `context.config["parameters"]`.
- Variations do not share mutable engine or portfolio state.
- Randomness is explicitly seeded so a selected combination reproduces its row.
- No winner is inferred before the user supplies selection criteria.

## Delivery note

State which dataset schema and libraries the script expects. List any remaining
data, execution, cost, liquidity, fill-quality, performance, or compatibility
limitations. A successful import or completed run alone does not prove that the
strategy faithfully models live trading.
