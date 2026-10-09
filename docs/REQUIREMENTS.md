# Portfolio Research & Execution Agent — Requirements

A planning document. Nothing here is implemented yet. Read it, argue with it, cut what you
do not want, and then we turn it into a staged build plan.

---

## 1. What we are building, in one paragraph

A system that continuously researches a set of companies you care about, maintains an
honest picture of your portfolio and its risks, and proposes trades backed by evidence.
Every proposal carries its reasoning, its sources, and what it would do to your risk.
You approve or reject it. On approval the system places the real order through your
broker's API, tracks the fill, reconciles your position, and later grades its own thesis
against what actually happened.

The system does not decide to trade on its own, and it does not tell you what to buy. It
gathers evidence, quantifies risk, and leaves the decision where it belongs.

---

## 2. The three execution modes

The same code path runs in three modes. Which mode you are in is configuration, not a
different implementation.

| Mode | Orders go to | Human approval | When you use it |
|---|---|---|---|
| **Paper** | An internal simulator | Not required | Default. Months of operation before anything is real. |
| **Approval-gated live** | The real broker API | Required, per order | Once the paper record justifies it. |
| **Unattended live** | — | — | Not built. See section 9. |

Paper mode must be genuinely realistic, meaning it simulates slippage, brokerage, STT,
stamp duty and the fact that your order moves nothing but still does not fill at the
price you saw. A paper record that assumes perfect fills teaches you nothing.

---

## 3. Vocabulary

A few terms used throughout, defined so the requirements are unambiguous.

- **Instrument** — a tradable thing, meaning a specific stock on a specific exchange.
- **Position** — how much of an instrument you hold, at what average cost.
- **Thesis** — the agent's written argument about an instrument at a point in time,
  including the case for, the case against, and a confidence level.
- **Proposal** — a concrete suggested order derived from a thesis, with quantity, order
  type and a risk impact assessment.
- **Point-in-time correctness** — the property that when the system reasons about a past
  moment, it can only see information that genuinely existed at that moment. This matters
  enormously for backtesting and is easy to get wrong.
- **Lookahead bias** — the bug where a backtest accidentally uses future information and
  therefore reports results that could never have been achieved. The single most common
  way trading systems fool their builders.

---

## 4. Functional requirements

Marked **[v1]** for the first working system, **[v2]** for later. Build v1 completely
before starting v2.

### 4.1 Universe and configuration

- **F1.1 [v1]** The user defines a watchlist of instruments the system tracks. This is
  explicit, not discovered, so that scope stays bounded and costs stay predictable.
- **F1.2 [v1]** The user configures risk limits: maximum position size as a percentage of
  portfolio, maximum sector concentration, maximum number of open positions, and maximum
  value traded per day.
- **F1.3 [v1]** The user configures which model provider and which model is used for each
  class of task.
- **F1.4 [v2]** The system can suggest additions to the watchlist based on screens the
  user defines, such as a sector or a size band.

### 4.2 Data ingestion

- **F2.1 [v1]** Ingest daily price history (open, high, low, close, volume) for every
  instrument in the watchlist, and keep it current.
- **F2.2 [v1]** Ingest corporate actions, meaning splits, bonuses and dividends, and adjust
  historical prices correctly. Getting this wrong silently corrupts every backtest.
- **F2.3 [v1]** Ingest quarterly results and annual reports for watchlist companies.
- **F2.4 [v1]** Ingest exchange announcements and filings for watchlist companies.
- **F2.5 [v1]** Ingest news from a configured set of sources, with each source carrying a
  quality rating, because a wire report and an anonymous blog are not equal evidence.
- **F2.6 [v1]** Every ingested fact is stored with two timestamps: when the fact became
  true, and when the system learned it. This is what makes point-in-time correctness
  possible later. Do not skip this in v1; retrofitting it is painful.
- **F2.7 [v1]** Ingestion is incremental and idempotent. Re-running it never duplicates
  data and never re-processes what is already stored.
- **F2.8 [v2]** Ingest earnings call transcripts and extract what management said, so that
  later quarters can be checked against earlier promises.

### 4.3 Research agents

- **F3.1 [v1] Fundamentals agent.** Reads filings and results for one company and reports
  what materially changed since the previous period, in plain language, with every claim
  citing the document and section it came from.
- **F3.2 [v1] News agent.** Reads recent news for one company, separates genuine new events
  from recycled commentary, resolves contradictions between sources by preferring higher
  quality ones, and reports what is actually new.
- **F3.3 [v1] Technical agent.** Computes standard indicators on the price series. It must
  report each indicator's historical hit rate on that instrument rather than asserting that
  a signal means anything. An indicator without a measured track record is decoration.
- **F3.4 [v1] Synthesis.** Combines the three into a thesis containing: the case for, the
  case against, the key uncertainty, and a calibrated confidence number. Every factual
  claim cites its source. A thesis with no case against it is rejected and regenerated,
  because an agent that only argues one side is not reasoning.
- **F3.5 [v1]** Research runs on a schedule per instrument, and can also be triggered on
  demand or by a significant event such as a results announcement.
- **F3.6 [v1]** Theses are stored permanently and never silently overwritten, so that the
  system's past opinions can be graded later.

### 4.4 Portfolio and risk

- **F4.1 [v1]** Maintain current holdings, average cost, realized and unrealized profit and
  loss, reconciled against the broker as the source of truth.
- **F4.2 [v1]** Compute concentration by instrument and by sector, and surface it plainly.
- **F4.3 [v1]** Compute correlation between holdings, so that four positions which are
  really one bet get identified as such.
- **F4.4 [v1]** Given a proposed order, simulate its effect on all of the above before it
  is shown to the user.
- **F4.5 [v1]** Enforce the configured risk limits as ordinary deterministic code, outside
  the model, in a layer the model cannot reach or argue with. A proposal that violates a
  limit is blocked and the reason is shown. This is a hard architectural requirement, not
  a preference.
- **F4.6 [v2]** Value-at-risk and drawdown estimates over the portfolio.

### 4.5 Proposals and approval

- **F5.1 [v1]** A proposal contains: the instrument, the action, the quantity, the order
  type and price, the thesis it derives from, the risk impact, and the reasons against.
- **F5.2 [v1]** Proposals are delivered to the user through a channel that works from a
  phone, so that approval is practical during market hours.
- **F5.3 [v1]** The user can approve, reject, or modify the quantity before approving.
- **F5.4 [v1]** A proposal expires if not acted on within a configured window, because an
  hour-old proposal is based on a stale price and must not silently execute later.
- **F5.5 [v1]** Rejections are recorded with an optional reason, and become training
  signal for evaluating the system later.
- **F5.6 [v1]** No order of any kind reaches a live broker without an explicit approval of
  that specific proposal. There is no bulk approval, no standing authorization, and no
  configuration flag that bypasses this.

### 4.6 Execution

- **F6.1 [v1]** Place orders through the broker API on approval, in live mode.
- **F6.2 [v1]** Track the full order lifecycle: placed, partially filled, filled, rejected,
  cancelled.
- **F6.3 [v1]** Order placement is idempotent. A retry after a timeout must never produce a
  second order. This requires a client-generated order identifier and a reconciliation
  step that checks the broker before retrying.
- **F6.4 [v1]** Handle partial fills correctly, meaning the position reflects what actually
  filled, not what was requested.
- **F6.5 [v1]** Reconcile the system's view of positions against the broker's view on a
  schedule, and raise an alert on any mismatch rather than silently trusting local state.
- **F6.6 [v1]** The paper simulator implements the same interface, including rejections and
  partial fills, so that the live path is exercised long before it is live.
- **F6.7 [v1]** A single switch halts all execution immediately, meaning a kill switch.

### 4.7 Measurement — the heart of the project

- **F7.1 [v1] Backtest harness.** Replay the strategy over historical data such that at
  each simulated moment the agent sees only what was known then. No exceptions, no
  convenient shortcuts.
- **F7.2 [v1] Lookahead detection.** An explicit test that deliberately tries to leak future
  data and confirms the harness refuses it. Without this you cannot trust your own results.
- **F7.3 [v1] Benchmark comparison.** Report performance against simply holding a Nifty
  index fund over the same period, including after costs. Report it prominently, including
  when it is unfavourable.
- **F7.4 [v1] Thesis grading.** Every thesis is revisited after a configured interval and
  graded against what actually happened.
- **F7.5 [v1] Calibration measurement.** When the system says it is seventy percent
  confident, measure how often it is right. Report the calibration curve.
- **F7.6 [v1] Agent-level evaluation.** A labelled set for the extraction tasks, meaning
  did the fundamentals agent correctly identify what changed, with precision and recall
  reported. This is separate from whether the trades made money.
- **F7.7 [v1]** Every run records tokens, cost, latency and which tools were called.

### 4.8 Pluggability

- **F8.1 [v1]** The user supplies their own model API key. Anthropic, OpenAI, Gemini and a
  local model through Ollama are all supported behind one interface.
- **F8.2 [v1]** The user supplies their own broker credentials. The broker integration sits
  behind an interface so a second broker can be added without touching the agents.
- **F8.3 [v1]** All credentials are encrypted at rest, never logged, and revocable.
- **F8.4 [v1]** The system's own capabilities, meaning portfolio state, thesis lookup and
  market data queries, are exposed as MCP servers, so any MCP client can use them.
- **F8.5 [v2]** Consume third-party MCP servers for additional data sources.

### 4.9 Transparency

- **F9.1 [v1]** Every proposal can be traced back through its thesis to the specific
  documents and data points that produced it.
- **F9.2 [v1]** A complete audit log of every decision, approval, order and outcome.
- **F9.3 [v1]** The user can correct a fact the system got wrong, and the correction
  persists.

---

## 5. Non-functional requirements

### 5.1 Correctness

- **N1.1** Point-in-time correctness in all historical reasoning. This is the highest
  priority non-functional property in the system, above performance and above elegance.
- **N1.2** Monetary values use exact decimal arithmetic. Never floating point for money.
- **N1.3** All timestamps are stored in UTC with explicit timezone handling at the edges.
  Market hours are IST; the gap between the two is a reliable source of bugs.
- **N1.4** Corporate action adjustments are verifiable against a known reference case.

### 5.2 Reliability and durability

- **N2.1** A crash mid-run never loses work and never repeats a side effect. Agent runs are
  checkpointed and resumable.
- **N2.2** Ingestion is at-least-once with idempotent writes, so duplicates are harmless.
- **N2.3** External API failures degrade gracefully. If news is unavailable, research
  continues and the thesis says news was unavailable, rather than silently omitting it.
- **N2.4** Network calls have timeouts, bounded retries with backoff, and a circuit breaker
  so a failing provider cannot stall the system.

### 5.3 Latency

- **N3.1** Research runs are offline work; minutes are acceptable.
- **N3.2** Once approved, order placement completes within a few seconds, because the price
  you approved is already becoming stale.
- **N3.3** Proposal delivery to the phone is near-immediate.

### 5.4 Security and privacy

- **N4.1** Broker credentials and model keys are encrypted at rest using the operating
  system keyring or an equivalent.
- **N4.2** Secrets never appear in logs, traces, prompts or error messages.
- **N4.3** The system runs locally on your machine. No portfolio data leaves it except the
  content of prompts sent to the model provider you chose.
- **N4.4** The broker session requires the daily manual authentication the broker mandates.
  Do not attempt to automate around it.
- **N4.5** The kill switch works even if the agent layer is wedged.

### 5.5 Cost

- **N5.1** A hard monthly spending ceiling on model usage, enforced in code, which stops
  the system rather than exceeding it.
- **N5.2** Cheap models for extraction and classification, strong models only for synthesis.
- **N5.3** Aggressive caching, since the same filing is read many times.
- **N5.4** Cost per research run is reported, so the economics stay visible.

### 5.6 Observability

- **N6.1** Every agent run is traced end to end, including each tool call, its inputs, its
  outputs and its duration.
- **N6.2** Structured logs with a correlation identifier that ties a proposal to its thesis
  to its order.
- **N6.3** Alerts on reconciliation mismatch, ingestion falling behind, and cost ceiling
  approach.

### 5.7 Testability

- **N7.1** Every external dependency sits behind an interface with a fake implementation,
  so the whole system runs offline in tests.
- **N7.2** Deterministic replay, meaning a recorded run can be re-executed identically.
- **N7.3** Model calls in tests are either recorded and replayed, or stubbed.

### 5.8 Legal and compliance

- **N8.1** The system presents evidence and risk. It does not issue buy, sell or hold
  recommendations to anyone other than the operator, and it is not distributed to others as
  advice. In India, distributing investment advice requires SEBI registration.
- **N8.2** Verify the current SEBI position on retail algorithmic trading and your broker's
  API terms before enabling live mode. These rules changed recently; check, do not assume.
- **N8.3** Respect the terms of service and rate limits of every data source.

---

## 6. High-level architecture

### 6.1 The layers

```
  Scheduler / event triggers
            │
            ▼
  ┌─────────────────────────────────────────┐
  │  Agent layer  (LangGraph)               │
  │  fundamentals · news · technical        │
  │  synthesis → thesis                     │
  └─────────────────────────────────────────┘
            │ tools (MCP)
            ▼
  ┌─────────────────────────────────────────┐
  │  Tool layer  (MCP servers)              │
  │  market data · filings · news           │
  │  portfolio · thesis store               │
  └─────────────────────────────────────────┘
            │
            ▼
  ┌─────────────────────────────────────────┐
  │  Risk engine   (deterministic code)     │  ← model cannot reach past here
  │  limits · concentration · simulation    │
  └─────────────────────────────────────────┘
            │
            ▼
  ┌─────────────────────────────────────────┐
  │  Proposal + approval  (human gate)      │
  └─────────────────────────────────────────┘
            │
            ▼
  ┌─────────────────────────────────────────┐
  │  Execution   paper │ live broker        │
  │  order lifecycle · reconciliation       │
  └─────────────────────────────────────────┘
```

The important structural idea: **the model lives entirely above the risk engine.** It can
propose anything; it cannot widen a limit, bypass a check, or reach the broker. The gate
below it is ordinary code that does not take instructions.

### 6.2 The data flow

1. Scheduled ingestion pulls prices, filings and news into storage, each fact stamped with
   when it became true and when we learned it.
2. A research run is triggered for an instrument, by schedule or by an event.
3. The three research agents work in parallel, each calling tools over MCP.
4. Synthesis combines their output into a thesis with sources and a confidence level.
5. If the thesis implies an action, a proposal is generated.
6. The risk engine evaluates the proposal, simulating its portfolio effect, and either
   blocks it with a reason or passes it.
7. The proposal is delivered for approval and waits, durably, possibly across a restart.
8. On approval the order goes to paper or to the broker, and its lifecycle is tracked.
9. Reconciliation confirms the resulting position matches the broker.
10. Later, the grader revisits the thesis and scores it against what happened.

### 6.3 Why multiple agents here is real

Not decoration. The three research agents have different inputs, different failure modes
and different evaluation criteria. The fundamentals agent reads long structured documents
and is judged on extraction accuracy. The news agent handles contradictory sources and is
judged on separating signal from recycling. The technical agent is pure computation whose
output is only meaningful alongside a measured hit rate. They genuinely do not belong in
one prompt, and they can run in parallel.

---

## 7. Tools and technology

Chosen with a bias toward fewer moving parts. Each line says what it is for and when to
introduce it.

### 7.1 Core

| Tool | Purpose | When |
|---|---|---|
| **Python 3.12+** | Everything. The agent and data ecosystem is Python. | Start |
| **uv** | Dependency and environment management. Fast, simple, now standard. | Start |
| **LangGraph** | Agent orchestration, checkpointing, human-in-the-loop interrupts. You already know it. | Start |
| **MCP Python SDK** | Building the tool servers. | Stage 2 |
| **Pydantic** | Typed structured output from models, and config validation. | Start |
| **PostgreSQL** | The one datastore: market data, filings, theses, proposals, orders, audit log. | Start |
| **SQLAlchemy + Alembic** | Data access and schema migrations. | Start |

A deliberate decision: **one database, not four.** Postgres can hold the time series, the
documents, the relational state and, with `pgvector`, the embeddings. Adding a specialized
time-series database or a separate vector store before you have felt the pain is complexity
for its own sake.

### 7.2 Data sources

| Tool | Purpose | Notes |
|---|---|---|
| **Broker historical API** | Prices and instrument master | Most accurate for Indian markets; you have it anyway |
| **NSE/BSE bhavcopy** | Daily official price and corporate action data | Free, authoritative, needs parsing |
| **yfinance** | Convenient price data for prototyping | Unofficial and sometimes wrong; do not rely on it for backtests |
| **BSE/NSE announcements** | Filings and corporate announcements | Free, messy |
| **RSS feeds of business press** | News | Start here before any paid news API |

### 7.3 Broker

| Tool | Purpose |
|---|---|
| **Zerodha Kite Connect**, or Upstox / Dhan / Angel One SmartAPI | Order placement, positions, historical data |

Build against your own `BrokerClient` interface from day one, with the paper simulator as
the first implementation. The real broker becomes the second implementation much later.

### 7.4 Approval channel

| Tool | Purpose |
|---|---|
| **Telegram Bot API** | Delivering proposals and receiving approvals |

This is the pragmatic choice given that you do not do frontend work. A Telegram message
with approve and reject buttons is a complete, usable interface that works from your phone
during market hours, and it takes an afternoon rather than a fortnight.

### 7.5 Supporting

| Tool | Purpose | When |
|---|---|---|
| **APScheduler** | Scheduled ingestion and research runs | Start |
| **Langfuse** (self-hosted) | Tracing agent runs, token and cost accounting | Stage 3 |
| **OpenTelemetry** | Structured traces and logs | Later |
| **pytest** | Tests, including the eval harness | Start |
| **Docker Compose** | Postgres and Langfuse locally | Start |
| **OS keyring** via `keyring` | Credential storage | Stage 4 |
| **Kafka / Redpanda** | Event-driven ingestion pipeline | Optional, stage 6 |

On Kafka: it is not needed for a single-user system, and APScheduler plus Postgres is
honestly sufficient. Introduce it only if you want the project to demonstrate the
event-driven durability you learned in the Kafka lab, and if you do, be prepared to explain
in an interview why it was a reasonable choice rather than a resume decoration.

### 7.6 Explicitly not using, for now

- A separate vector database. `pgvector` is enough at this scale.
- A dedicated time-series database. Postgres handles daily bars for a few hundred
  instruments without noticing.
- A web frontend. Telegram plus a command line covers it.
- A workflow engine such as Temporal. LangGraph's checkpointing covers the agent runs.
- Fine-tuning. Nothing here needs it.

---

## 8. What to read before implementing

Roughly in the order you will need it.

**Agents and orchestration**
- LangGraph persistence and checkpointing: how state survives a restart.
- LangGraph interrupts: the mechanism behind the human approval gate. This is the piece
  you have not used yet and it is central to this design.
- Structured output with Pydantic, so agent results are typed rather than parsed from prose.

**MCP**
- The protocol's model of tools, resources and prompts, and why it separates them.
- Building a server with the Python SDK, and connecting a client to it.
- What makes a tool description good, since the model's reliability depends heavily on it.

**Market data correctness**
- Corporate action adjustment, meaning what a split or bonus does to a historical series.
- Point-in-time databases and the idea of bitemporal data, which is the two-timestamp
  requirement in F2.6.
- Lookahead and survivorship bias, meaning the two ways backtests lie.

**Trading mechanics**
- Order types and the order lifecycle.
- Slippage, brokerage, STT and the real cost of a round trip in India.
- Idempotency in order placement, which is a distributed systems problem you already know
  from a different direction.

**Evaluation**
- Calibration, meaning what it means for a confidence number to be honest.
- Precision and recall for the extraction tasks.
- Why benchmark comparison after costs is the only honest performance measure.

**Regulatory**
- The current SEBI framework for retail algorithmic trading.
- Your broker's API terms, particularly around automated order placement and session
  authentication.

---

## 9. Explicitly out of scope

- **Unattended live trading.** The system never places a live order without a specific
  human approval. This is a design decision, not a missing feature, and it is the reason
  the architecture puts the risk engine below the model.
- **Advice to other people.** Operator-only. Distribution would require SEBI registration.
- **Intraday or high-frequency strategies.** The design assumes daily bars and a horizon of
  weeks or longer. Sub-second concerns would change every decision in this document.
- **Derivatives.** Equity only in v1.
- **Multi-user or hosted operation.** Single user, local.
- **Tax computation.** Record-keeping only.

---

## 10. Open decisions for you

1. **Broker.** Which one do you actually have an account with? It determines the API.
2. **Watchlist size.** Twenty instruments or two hundred? It changes the cost profile.
3. **Horizon.** Weeks, or months? It determines what data matters and how long before you
   have enough evidence to judge the system.
4. **Kafka.** Include it for the durability story, or keep the stack minimal?
5. **How long in paper mode** before considering live. My suggestion is at least three
   months and at least thirty graded theses, but it is your money and your call.

---

## 11. What the first stage looks like

Not the full plan, just the shape, so you can see where this goes:

- **Stage 1** — Data foundation. Ingest prices and corporate actions for a small watchlist,
  with the two-timestamp model in place. No agents at all. Prove the data is correct,
  because everything downstream rests on it.
- **Stage 2** — One agent, one tool server. The fundamentals agent reading filings through
  an MCP server, producing a typed thesis with citations.
- **Stage 3** — The remaining research agents and synthesis, running in parallel in
  LangGraph.
- **Stage 4** — Risk engine, proposals, and the Telegram approval gate, with the paper
  simulator behind it.
- **Stage 5** — The backtest harness and the lookahead test. The hardest stage, and the one
  that produces the numbers you will put on your resume.
- **Stage 6** — Thesis grading and calibration measurement.
- **Stage 7** — Real broker integration, order lifecycle, reconciliation, kill switch.

Stage 5 before stage 7 is deliberate. You should know whether the thing works before you
connect it to money.
