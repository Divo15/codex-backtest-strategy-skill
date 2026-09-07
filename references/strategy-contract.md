# Trade-log Analytics Dashboard Strategy Contract v2

## Required module interface

Every generated script must expose this interface:

```python
STRATEGY_CONTRACT_VERSION = "2"
RUN_MODE = "single"  # or "sweep"
SWEEP_PARAMETER_SETS = ()


def run_strategy(context):
    ...
```

Importing the module must only define declarations, functions, and classes. It
must not run the strategy, inspect the filesystem, load market data, contact a
service, install packages, request input, or create output files.

## Dashboard context

The dashboard passes an object with three fields:

| Field | Meaning |
| --- | --- |
| `context.run_id` | Unique identifier owned by the dashboard. Use it unchanged in every returned trade. |
| `context.market_data` | `pathlib.Path` for the dataset selected in the dashboard. |
| `context.config` | Run configuration containing `instrument`, `period`, `execution`, and `parameters`. |

The script must read market data only through `context.market_data`. Never
embed a developer's local data path. The dashboard detects and supplies the
available period. Strategy parameters belong in
`context.config.get("parameters", {})`; make a local copy before modifying it.

The strategy owns trading rules and defaults such as lot size, capital model,
entry time, expiry selection, targets, stops, re-entry, and hedging. Execution
values supplied in `context.config` must not be silently replaced.

## Successful result

`run_strategy(context)` returns:

```python
{
    "completed_trades": completed_trade_rows,
    "completed_trade_count": len(completed_trade_rows),
    "trade_mapper": None,             # optional when rows are already canonical
    "equity_snapshots": snapshots,    # optional
    "metadata": {                     # optional and descriptive only
        "strategy_name": "...",
        "engine": "...",
    },
}
```

`completed_trades` must be the engine's authoritative collection of actually
closed trades or closed legs. A pandas DataFrame must be normalized with
`to_dict(orient="records")`; ordinary DataFrame iteration yields column names.
Capture the authoritative count before normalization and ensure it remains the
same. A valid no-trade run returns an empty collection and count `0`.

Let exceptions propagate. Never catch an error and return invented or partial
success data.

## Canonical closed-trade fields

Each row represents one completed trade or one closed leg:

| Field | Required | Rule |
| --- | --- | --- |
| `run_id` | Yes | Exactly `context.run_id`. |
| `trade_id` | Yes | Non-empty and unique within the run. |
| `batch_id` | No | Same value for legs belonging to one position or structure. |
| `leg_id` | No | Identifies a leg within a batch. |
| `strategy` | Yes | Stable descriptive strategy name. |
| `symbol` | Yes | Instrument or contract actually traded. |
| `side` | Yes | Exactly `LONG` or `SHORT`. |
| `entry_time` | Yes | ISO 8601 or datetime with a timezone offset. |
| `exit_time` | Yes | ISO 8601 or datetime with a timezone offset; not before entry. |
| `quantity` | Yes | Positive actual quantity. |
| `entry_price` | Yes | Positive actual fill price. |
| `exit_price` | Yes | Positive actual fill price. |
| `multiplier` | No | Positive contract multiplier; defaults to `1`. |
| `fees` | No | Total non-negative fees for that row; defaults to `0`. |

Do not add `pnl`, `gross_pnl`, `net_pnl`, `return`, `drawdown`, win rate,
Sharpe, or other derived metrics. The dashboard reconstructs those values from
raw executions. Do not label bar closes as fills unless that is the explicit
fill model; disclose the actual fill assumption.

For a multi-leg structure, return every closed leg. Use one `batch_id` for the
whole structure and a unique `trade_id` and `leg_id` for each leg. Use actual
contract symbols when available.

## Equity snapshots

Return `equity_snapshots` only when the engine captured contemporaneous account
or position valuations during the run. Never derive an intraday curve by
interpolating between entry and exit or from final trade P&L.

Each snapshot mapping contains:

```python
{
    "timestamp": "2026-01-02T09:20:00+05:30",
    "realized_pnl": 0.0,
    "unrealized_pnl": -1250.0,
}
```

Timestamps must be timezone-aware, unique, and strictly increasing. Start with
a zero snapshot before the first entry. Finish after every position is closed,
with zero unrealised P&L and cumulative realised P&L equal to the reconstructed
trade-log result after fees. Capture marks at every engine bar or tick and
after executions; sparse sampling can miss the true worst drawdown.

## Sweeps

For a single run:

```python
RUN_MODE = "single"
SWEEP_PARAMETER_SETS = ()
```

For a sweep:

```python
RUN_MODE = "sweep"
SWEEP_PARAMETER_SETS = (
    {"take_profit": 0.20, "stop_loss": 1.5},
    {"take_profit": 0.25, "stop_loss": 1.5},
)
```

Declare 1–25 unique JSON-serializable mappings. The dashboard calls
`run_strategy(context)` once per mapping and places that exact mapping in
`context.config["parameters"]`. The number of parameter names is unrestricted;
the limit applies to total combinations. Validate unknown keys instead of
ignoring them. Keep all results independent and deterministic.

## Package and output ownership

Import dependencies normally. Package installation belongs to the dashboard's
`.venv`, not inside the strategy. The dashboard worker creates and validates
`trades.csv`, `equity.csv`, checksum manifests, and analytics. Strategy code
must not write those files or calculate dashboard metrics.
