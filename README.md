# Portfolio Research & Execution Agent

A multi-agent system that researches a watchlist of companies, maintains an honest picture
of portfolio risk, and proposes trades backed by cited evidence. Every proposal is reviewed
and approved by a human before any real order is placed.

**Status: design stage.** Nothing is implemented yet. The requirements are written; the
build has not started.

---

## What it does

- Ingests prices, corporate actions, filings and news for a defined watchlist, recording
  both when each fact became true and when the system learned it.
- Runs three research agents in parallel — fundamentals, news and technical — each with its
  own tools, failure modes and evaluation criteria.
- Synthesises their output into a **thesis**: the case for, the case against, the key
  uncertainty, and a calibrated confidence level, with every claim citing its source.
- Simulates the effect of any proposed trade on portfolio concentration and correlation
  before showing it to you.
- Delivers proposals for approval, places the approved order, tracks the fill, and
  reconciles the resulting position against the broker.
- Grades its own past theses against what actually happened, and measures whether its
  stated confidence is honest.

## What it deliberately does not do

- **It does not trade unattended.** No live order is placed without an explicit human
  approval of that specific proposal. There is no bulk approval and no bypass flag.
- **It does not give investment advice.** It presents evidence and quantified risk. The
  decision stays with the operator. It is not distributed to anyone else as advice.
- **It does not try to automate around broker authentication.** The daily manual login the
  broker requires is honoured.

## The structural idea

The model sits entirely above the risk engine, and cannot reach past it.

```
agents  →  propose anything
   │
risk engine (deterministic code, no model in the loop)
   │           limits · concentration · simulation
   ▼
human approval
   │
   ▼
execution  (paper by default, broker only after approval)
```

A model can argue with a prompt. It cannot argue with a function that returns False.

## Planned stages

| Stage | Goal |
|---|---|
| 1 | Data foundation — ingestion, corporate actions, two-timestamp model. No agents. |
| 2 | One research agent reading filings through an MCP server, producing a typed thesis. |
| 3 | Remaining research agents and synthesis, in parallel via LangGraph. |
| 4 | Risk engine, proposals, approval gate, paper execution. |
| 5 | Backtest harness with point-in-time correctness and an explicit lookahead test. |
| 6 | Thesis grading and calibration measurement. |
| 7 | Live broker integration — order lifecycle, reconciliation, kill switch. |

Stage 5 comes before stage 7 on purpose. Find out whether the system works before
connecting it to money.

## Stack

Python · LangGraph · MCP · PostgreSQL (with `pgvector`) · Telegram for approvals ·
Langfuse for tracing. One database rather than four; no web frontend.

## Documentation

- [Requirements](docs/REQUIREMENTS.md) — functional and non-functional requirements,
  architecture, tool choices, and reading list.

## Disclaimer

Personal project, operated by its author on their own account. Nothing here is investment
advice. Automated trading in India is subject to SEBI regulation and to broker API terms;
anyone reusing this is responsible for their own compliance.
