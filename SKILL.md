---
name: dashboard-strategy-author
description: Generate, adapt, or review Python backtest strategies for the Trade-log Analytics Dashboard Strategy Contract v2. Use when a script must run from the dashboard, consume its selected dataset, support single or sweep execution, and return auditable completed trades and optional mark-to-market snapshots.
---

# Dashboard strategy author

Create scripts that the local Trade-log Analytics Dashboard can execute without
guessing how to supply data or retrieve trades.

Before writing or adapting a script, read
[references/strategy-contract.md](references/strategy-contract.md). Read
[references/examples.md](references/examples.md) when a working single-run or
sweep skeleton would help.

Preserve the requested trading behavior exactly. Keep indicators, entries,
exits, re-entries, sizing, hedges, stops, targets, and fill assumptions in the
strategy. If a required rule or data field is unknown, state what is missing;
do not silently invent or simplify it.

Generate an import-safe module that:

- declares `STRATEGY_CONTRACT_VERSION = "2"`;
- declares `RUN_MODE` as `"single"` or `"sweep"`;
- exposes `run_strategy(context)`;
- reads market data only from `context.market_data`;
- reads dashboard and sweep inputs from `context.config`;
- returns the authoritative closed trades and their exact count;
- maps every closed leg to raw execution fields, without calculated P&L or
  analytics fields; and
- returns engine-observed equity snapshots when true intraday and unrealised
  drawdown is required.

For sweeps, declare the exact `SWEEP_PARAMETER_SETS` mappings requested by the
user. The dashboard supports at most 500 variations and replaces
`context.config["parameters"]` for each variation. Make every declared key
affect the strategy or reject it clearly.

When the user supplies value ranges for several parameters, build the complete
Cartesian product so no requested combination is skipped. Do not calculate a
score or label a winner in strategy code; the dashboard ranks profitable
combinations from independently calculated execution metrics. Make repeated
execution deterministic: the dashboard discards each sweep variation's bulky
artifacts, automatically reruns the recommended combination for full analytics,
and verifies that its core metrics match the sweep row. The user can return to
the comparison table and rerun a different combination. Only the automatically
recommended result is stored in the dashboard's persistent Best history.

Do not start a backtest, read files, download data, install packages, prompt for
input, or write dashboard output while the module is imported. Do not write
`trades.csv`, `equity.csv`, manifests, or analytics from the strategy; the
trusted dashboard worker owns those outputs.

Before delivering a script, review it with
[references/review-checklist.md](references/review-checklist.md). Explain any
remaining engine, data, fill-quality, cost, or snapshot limitation plainly.
