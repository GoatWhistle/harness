# Architecture

How MANDATE turns market data into paper orders, and which code owns each decision. Runtime evidence for every claim is in the [verification report](MANDATE_VERIFICATION.md).

## Trust zones

1. **Research plane — unprivileged.** Parses bounded market/news inputs, calculates features and produces evidence. It has no trading client.
2. **Agent plane — creative but untrusted.** TrueForge and the model can research, challenge a thesis and propose an action. Tool allowlists prevent direct execution.
3. **Execution plane — deterministic.** The guard owns paper broker credentials, reloads the human mandate, fetches fresh state, applies every limit with `Decimal`, and journals the result.

## What the agent understands

The production funnel monitors 18 liquid AI/platform/infrastructure equities plus SPY:

`AAPL` `MSFT` `NVDA` `GOOGL` `AMZN` `META` `AMD` `AVGO` `ORCL` `IBM` `PLTR` `CRM` `ANET` `TSM` `ASML` `ARM` `BABA` `BIDU` `SPY`

It turns price, liquidity, market regime and attributable news into eight explainable strategy outputs:

- price momentum and volatility-adjusted momentum;
- rolling z-score and RSI mean reversion;
- volume-confirmed breakout;
- MACD trend;
- structured LLM news impact confirmed by price;
- regime-weighted ensemble.

News is scored as structured data — `sentiment`, `confidence`, `event_type`, `horizon`, `novelty_vs_48h`, affected tickers and reason — rather than by a tiny positive/negative word list. The LLM enriches evidence; deterministic code still owns the decision boundary.

ATR14 sizing converts risk budget and signal strength into a whole-share quantity, capped by mandate headroom and same-side correlation clusters. SPY trend/range state changes ensemble weights and scales gross risk. Completed 5/15/60-minute outcomes update bounded strategy multipliers, including counterfactual outcomes for parked ideas: adaptive behavior without letting a model rewrite policy.

## Safety invariants

| Invariant | Enforcement |
|---|---|
| **Paper only** | Exact host allowlist accepts `https://paper-api.alpaca.markets`; live, HTTP, credential-bearing and look-alike URLs are rejected |
| **No hidden authority** | The agent cannot write the mandate and cannot access a direct broker execution tool |
| **Fresh-state validation** | Account, broker clock, latest trade, pending orders and every limit are re-read immediately before submit |
| **Fail closed** | Missing/malformed policy, transport errors, stale data, closed sessions and unknown rules deny execution |
| **Approval is not an override** | Human approval resumes the tool call; the guard repeats policy checks independently |
| **Bounded risk** | Position, gross exposure, daily loss, order count, session, universe, instrument and order type are deterministic |
| **Safe retries** | Canonical intent fingerprints, stable client IDs and durable provenance prevent duplicate or mutated retries |
| **Auditable changes** | Every prepared, denied and submitted event carries the SHA-256 fingerprint of the validated mandate |
| **Protected exits** | Cancellation requires submitted provenance; closes are risk-reducing and explicitly opt-in |
| **Untrusted news** | Input is size-bounded, normalized, timestamped and treated only as data—not as instructions |

The guard supports human-authored predecisions such as `daily_loss_pct >= 1 → park_new_orders`. Unknown metrics or actions make the mandate invalid instead of being guessed.

## TrueForge: the agent harness

TrueForge provides the durable agent runtime around the safety kernel:

- persistent sessions, tool-call traces and context compaction;
- MCP tool isolation and explicit execution allowlists;
- approval pauses for irreversible actions;
- dynamic read-only subagents for price research, news research and risk challenge;
- sandbox access for bounded experiments, watched by a persisted-event policy auditor;
- one operator workspace for autonomous runs and conversational control.

The background runner consumes Alpaca news and market WebSockets with a configurable REST fallback. Every cycle must end in `ACTION: PARK` or `ACTION: PROPOSE`; a mechanical post-model gate parks any proposal that violates session, liquidity, staleness, relative-volume, macro or mandate constraints.

## Example mandate

Human authority is small enough to read and strict enough to execute:

```yaml
universe: [AAPL, MSFT, NVDA, GOOGL, AMZN, META, AMD, AVGO, SPY]
instruments: [equity]
order_types: [limit]
session: regular_hours_only
limits:
  max_position_pct: 10
  max_gross_exposure_pct: 60
  max_daily_loss_pct: 2
  max_orders_per_day: 20
predecided:
  - when: daily_loss_pct >= 1
    then: park_new_orders
    reason: Protect the remaining daily loss budget before the hard stop.
allow_short_positions: false
allow_risk_reducing_market_close: true
expires: 2099-08-28T20:00:00Z
```

See the complete [example mandate](../mandate/mandates/example.yaml).

## Product experience

### One decision at a time

The operator sees a compact decision card, ready position size, order terms, rationale, and the exact safety checks that passed. Approve or deny; the completed card moves into the journal instead of taking over the screen.

### A live, legible portfolio

Equity, P&L, exposure, positions, orders, news, agent decisions and mandate headroom update on one full-width dashboard. News has a focused latest-story view and a separate feed; monitoring controls live in an IDE-style settings drawer.

![MANDATE dynamic paper portfolio and audit journal](assets/mandate-live-portfolio.png)

### Chat as the control plane

The agent keeps working outside chat. Chat remains available to explain a decision, inspect evidence, adjust the thesis, narrow the symbol pool, or propose a cadence/risk-posture change. Persistent trajectory changes require explicit confirmation and cannot expand the hard mandate universe.
