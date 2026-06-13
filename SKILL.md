---
name: ai-trading-agent
description: Standard Operating Procedure for US-equity trading via the Robinhood Agentic Trading MCP, driven by two independent cluster signals — congressional buys (Signal A) and corporate insider open-market purchases (Signal B). Use whenever the user asks to scan for trade signals, generate a daily trade list, place orders, review the trade log, or run the daily SOP. Defaults to paper mode; live trading requires explicit configuration.
---

# AI Trading Agent SOP

## 1. When to invoke

Activate this skill when the user asks any of:
- "Run the SOP" / "scan today's clusters" / "what should I trade today"
- "Check Form 4 filings" / "any politician buys today"
- "Place this trade" / "execute on signal X" / "review trade log"
- Scheduled / cron-style daily cadence requests

## 2. Non-negotiable rules (apply always, override everything)

These are bright-line invariants. If any rule conflicts with a signal score, the rule wins.

1. **Never invest more than 5% of total portfolio value in a single position.**
2. **Never place a market order.** Limit orders only, within 0.2% of the current ask.
3. **If an open position drops 8% from your entry price, close it without waiting** — no second-guessing, no averaging down.
4. **Always write a journal entry for the day, even on days with zero trades.** No-trade journal entries record: signals scanned, candidates skipped, why.
5. **Never place trades when the US market is closed.** Check market status via the Robinhood MCP (or fall back to calendar: regular session Mon–Fri 09:30–16:00 ET excluding US market holidays). No extended-hours orders.

Full mechanics in `references/rules.md` §0.

## 3. Hard preconditions (verify in this order, halt on failure)

1. **MCP connected.** Run `tools/list`. Confirm Robinhood trading tools are present. If not, tell the user the MCP is unreachable and stop.
2. **Account is the dedicated Agentic account.** Read the account identifier from the MCP. Confirm it is the Agentic account, **not** the user's primary individual account. Robinhood only permits agentic trading from dedicated Agentic accounts.
3. **Market is open.** Confirm regular session is live (rule §5 above). If closed, signal scans continue but no order tool may be invoked.
4. **`mode.toml` exists and is parseable.** Read `mode.toml` at repo root. Required fields: `mode` (`"paper"` or `"live"`), `live_allowlist` (list, may be empty), `require_manual_confirm` (bool). If the file is missing, create it with paper defaults and tell the user.
5. **`watchlist.json` exists and is parseable.** Required fields: `watchlist` (array; `[]` allowed) and `cash_reserve_pct` (number 0–100). Each watchlist entry needs `symbol`, `description`, `max_allocation_pct`. If missing, halt and tell the user.
6. **No kill switch.** If `KILL_SWITCH` file exists at repo root, refuse to place any order (signal scans still allowed).
7. **`trade-log.jsonl` is writable.** Append a no-op heartbeat line if needed to prove writability. **No order tool may be invoked unless the trade log is writable.**

## 4. Daily SOP loop

```
┌─ 1. Refresh Signal A (politicians)
│     load: references/politician-signal.md (scoring), references/data-sources.md (feed)
│
├─ 2. Refresh Signal B (insiders)
│     load: references/sec-edgar-form4.md (XML + discovery)
│     load: references/insider-signal.md (scoring)
│
├─ 3. For each candidate ticker, compute conviction tier
│     load: references/rules.md (sizing, exits, combined-signal bonus)
│
├─ 4. Run the 5-question Decision Framework per candidate (see §5)
│     Any unsatisfactory answer → skip the trade, log the reason.
│
├─ 5. Emit a pre-trade review block per surviving candidate
│     ticker | signal_source | conviction | tier | qty | limit | max_loss
│
├─ 6. WRITE PENDING ENTRY to trade-log.jsonl  ← BEFORE any MCP order tool
│
├─ 7. Execute based on mode:
│     paper → mark result: "paper", skip MCP order tool
│     live  → call MCP order tool; if require_manual_confirm=true, pause
│             update log entry with mcp_response + result
│
├─ 8. Scan all open positions for the −8% hard stop (rule §3). Close immediately on breach.
│
└─ 9. Update journal/{YYYY-MM-DD}.md (human-readable markdown — template in rules §7)
     MANDATORY even on zero-trade days (rule §0.4).
```

## 5. Decision Framework (answer all 5 before every trade)

For each candidate that survives signal scoring, the agent **must** answer all five questions out loud and log the answers to the journal entry. Any unsatisfactory answer kills the trade.

1. **What is the current portfolio cash balance?** Read from the Robinhood MCP. Trade size must fit within available cash **and respect the `cash_reserve_pct` floor** from `watchlist.json` (e.g. with `cash_reserve_pct = 20`, deployed capital may not exceed 80% of equity).
2. **What positions are already open?** Read from the MCP. Block if this ticker is already at its per-symbol cap (watchlist value or 5% default), if it isn't in a populated watchlist, or if total cluster exposure ≥ 25%.
3. **What does recent news say about this ticker?** Pull a quick news read (Robinhood MCP if available, otherwise a brief web search). Block if there's an active going-private, bankruptcy, fraud, SEC enforcement, or accounting-restatement headline in the last 14 days.
4. **What do the 20-day and 50-day moving averages tell you?** Compute or fetch. Prefer trades where the close is above both MAs (constructive trend); if the close is below both, downgrade conviction by one tier; if 20-day is sharply below 50-day with a wide gap, skip.
5. **What is the risk if this trade goes wrong?** State the dollar loss at the 8% hard stop (`qty × entry × 0.08`). Confirm that loss is within the daily loss cap (2% of equity) after summing with any already-realized intraday losses.

## 6. Decision pointers (load on demand)

| When you need… | Load this file |
| --- | --- |
| Where any data feed lives (politician, insider, social, quotes) | `references/data-sources.md` |
| How to discover & parse SEC Form 4 XML | `references/sec-edgar-form4.md` |
| Signal A scoring (politician clusters) | `references/politician-signal.md` |
| Signal B scoring (insider clusters) | `references/insider-signal.md` |
| Signal C scoring (social media — **draft, not yet active**) | `references/social-signal.md` |
| Mode, watchlist, sizing, exits, kill switch, trade-log schema | `references/rules.md` |

Do **not** preload reference files unless step 1–7 needs them. Progressive disclosure keeps context lean.

## 7. Logs — two artifacts, both mandatory

The SOP writes two files every day. They serve different purposes; neither replaces the other.

**`trade-log.jsonl`** (repo root, append-only, machine-readable). One JSON line per order state transition. Written **before** the MCP order tool is invoked — this is the SOP's core auditability guarantee. It is also the **single analytics source**: every trade across every day lives here, so a single jq/DuckDB/pandas read covers the whole history. Full schema and analytics recipes in `references/rules.md` §7.

**`journal/{YYYY-MM-DD}.md`** (human-readable markdown, one file per day). Sections: Portfolio Status → Market Research → Trades Executed → Positions Closed → End-of-Day Reflection. Template in `references/rules.md` §7. Mandatory every trading day, including no-trade days.

## 8. Stop conditions

Halt new entries and tell the user when any of these hit:
- Market is closed (rule §5).
- Daily loss cap breached (see `references/rules.md`).
- Signal freshness > 24h (re-pull failed).
- MCP unreachable mid-loop.
- `KILL_SWITCH` file appears.
- `mode.toml` becomes unparseable.

## 9. Safety defaults (overridden only by explicit user instruction)

- Default mode: **paper**.
- Default order side: **buy** only. Sells happen only via exit rules in `references/rules.md` — never on a Signal-A or Signal-B firing.
- Default asset class: **US-listed common equity**. No options, no crypto, no OTC.
- Default order type: **limit**, never market.
- No leverage, no shorting.
- No extended-hours / pre-market / after-hours orders.

## 10. Reporting back to the user

After each daily loop, summarize in 5 lines or fewer:
- Candidates surfaced (count by tier).
- Orders placed (count + mode).
- Open positions touched (incl. any −8% stop-outs).
- Stop conditions hit, if any.
- Path to today's `trade-log.jsonl` and `journal/{YYYY-MM-DD}.md`.
