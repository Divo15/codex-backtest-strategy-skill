# Compatible strategy examples

These skeletons show the integration boundary. Replace the placeholder engine
calls with the user's complete strategy without changing its rules.

## Single run

```python
from zoneinfo import ZoneInfo

STRATEGY_CONTRACT_VERSION = "2"
RUN_MODE = "single"
SWEEP_PARAMETER_SETS = ()


def _map_closed_leg(leg, index, run_id):
    return {
        "run_id": run_id,
        "trade_id": str(leg.id),
        "batch_id": str(leg.position_id),
        "leg_id": str(leg.leg_id),
        "strategy": "protected-straddle",
        "symbol": leg.symbol,
        "side": leg.side.upper(),
        "entry_time": leg.entry_time,
        "exit_time": leg.exit_time,
        "quantity": leg.quantity,
        "entry_price": leg.entry_price,
        "exit_price": leg.exit_price,
        "multiplier": leg.multiplier,
        "fees": leg.total_fees,
    }


def run_strategy(context):
    parameters = dict(context.config.get("parameters", {}))
    period = dict(context.config.get("period", {}))

    engine = build_engine(
        market_data=context.market_data,
        start_date=period.get("start_date"),
        end_date=period.get("end_date"),
        parameters=parameters,
    )
    engine.run()

    authoritative_legs = list(engine.closed_legs)
    rows = [
        _map_closed_leg(leg, index, context.run_id)
        for index, leg in enumerate(authoritative_legs)
    ]
    if len(rows) != len(authoritative_legs):
        raise RuntimeError("Closed-leg normalization changed the engine count")

    snapshots = [
        {
            "timestamp": item.timestamp,
            "realized_pnl": item.realized_pnl,
            "unrealized_pnl": item.unrealized_pnl,
        }
        for item in engine.valuation_history
    ]

    return {
        "completed_trades": rows,
        "completed_trade_count": len(authoritative_legs),
        "equity_snapshots": snapshots,
        "metadata": {
            "strategy_name": "protected-straddle",
            "engine": type(engine).__name__,
        },
    }
```

`build_engine` and its trading logic are intentionally unspecified. Use the
actual engine and authoritative closed-trade collection.

## Sweep

```python
STRATEGY_CONTRACT_VERSION = "2"
RUN_MODE = "sweep"
SWEEP_PARAMETER_SETS = (
    {"take_profit": 0.20, "stop_loss": 1.25, "entry_time": "09:20"},
    {"take_profit": 0.25, "stop_loss": 1.25, "entry_time": "09:20"},
    {"take_profit": 0.30, "stop_loss": 1.50, "entry_time": "09:30"},
)

ALLOWED_PARAMETERS = {"take_profit", "stop_loss", "entry_time"}


def run_strategy(context):
    parameters = dict(context.config.get("parameters", {}))
    unknown = set(parameters) - ALLOWED_PARAMETERS
    if unknown:
        raise ValueError(f"Unsupported sweep parameters: {sorted(unknown)}")

    engine = build_engine(
        market_data=context.market_data,
        parameters=parameters,
        period=context.config.get("period", {}),
    )
    engine.run()
    closed_legs = list(engine.closed_legs)
    rows = [map_leg(leg, i, context.run_id) for i, leg in enumerate(closed_legs)]
    return {
        "completed_trades": rows,
        "completed_trade_count": len(closed_legs),
        "equity_snapshots": list(engine.valuation_history),
    }
```

The dashboard supplies a different `context.run_id` and parameter mapping for
each variation. Do not create a grid inside `run_strategy`; declare the exact
combinations in `SWEEP_PARAMETER_SETS`.
