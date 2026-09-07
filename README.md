# Codex skill for dashboard-compatible backtest strategies

This repository contains an installable Codex skill for generating and
reviewing Python strategies that run in the Trade-log Analytics Dashboard.

The skill teaches Codex the dashboard's Strategy Contract v2, including:

- the `run_strategy(context)` entry point;
- dashboard-supplied datasets and configuration;
- authoritative completed-trade output;
- raw trade fields used for independent analytics;
- observed intraday equity snapshots; and
- parameter optimization with up to 500 combinations and explicit user-defined
  winner selection.

## Install in Codex

Ask Codex:

> Install the skill from
> https://github.com/Divo15/codex-backtest-strategy-skill

After installation, restart Codex if it does not immediately appear in the
skills list.

## Use it

Ask Codex to generate or adapt a strategy for the Trade-log Analytics
Dashboard. You can also invoke it explicitly:

> Use `$dashboard-strategy-author` to convert this strategy into a compatible
> dashboard script without changing its trading rules.

The skill standardizes the execution boundary. It does not define or improve a
trading strategy, guarantee data quality, or guarantee profitable results.
