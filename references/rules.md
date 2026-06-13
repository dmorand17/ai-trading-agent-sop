# Trading Rules

Authoritative reference for **mode handling, position sizing, entries, exits, kill switches, combined-signal interactions, and the mandatory trade-log schema.**

Everything in this file is tunable **except §0**. Treat `mode.toml` and the knobs in `references/politician-signal.md` / `references/insider-signal.md` as the single sources of truth at runtime.

---

## 0. Non-negotiable rules (these always win)

If anything in §1–§9 conflicts with §0, **§0 wins**. These are bright-line invariants.

### 0.1 Position cap: 5% per single position (overridable per-symbol via watchlist)

- **Default cap: 5% of total portfolio value per single ticker.** Exposure = `(open_qty × current_mark) / total_equity`. Includes pre-existing holdings, not just cluster-signal positions.
- **Per-symbol override:** if the ticker is in `watchlist.json` (§2) with `max_allocation_pct`, that value **replaces** the 5% default for that ticker. Watchlist values may be higher or lower than 5% — the user has explicitly sanctioned the level.
- **Cash reserve interacts:** even with a higher per-symbol cap, total deployed capital still respects `cash_reserve_pct` (§2.3) — you cannot push cash below the reserve floor regardless of any single-position cap.
- The non-negotiable invariant is "there is *always* an enforced per-position cap and a cash reserve floor." The watchlist sets the levels; the SOP may never trade past them.

### 0.2 Order type: limit only, within 0.2% of ask

- Never `market`. Never `stop_market`. Never `market-on-close`.
- Entry limit = `current_ask × 1.002`.
- Exit limit = `current_bid × 0.998`.
- If the spread is so wide that `current_ask × 1.002` would still be inside the spread, skip the trade — the book is too thin.

### 0.3 Hard stop: −8% from entry, close without waiting

- For every open position, compute `(current_mark − entry_price) / entry_price` continuously during the SOP loop.
- If ≤ `−0.08`, submit a closing limit at `current_bid × 0.998` **immediately**, on the same loop pass.
- No averaging down, no "wait for the bounce," no overriding by signal score.
- This is independent of, and tighter than, the trailing stop in §4.3. The 8% hard stop catches early failures; the trailing stop catches reversals from a profitable peak.

### 0.4 Journal mandatory every day, even on no-trade days

- A human-readable `journal/{YYYY-MM-DD}.md` file must be created/updated on every SOP invocation, **whether or not any orders were placed**.
- Template and required sections are defined in §7 (Daily Journal — Markdown Format).
- On a no-trade day, fill the "Trades Executed" section with `None today.` and still complete Portfolio Status, Market Research, Positions Closed, and End-of-Day Reflection. The reflection should record *why* no trades fired — that's the data for tuning the SOP.
- The journal is the **human-facing** record. It is distinct from `trade-log.jsonl` (machine-facing audit log, §7). Both are mandatory.

### 0.5 Never trade when the market is closed

- Before any order tool is invoked, confirm regular US market session is currently open.
- Source of truth, in order:
  1. Robinhood MCP market-status read (if exposed by `tools/list`).
  2. Calendar fallback: Mon–Fri, 09:30–16:00 America/New_York, excluding US market holidays (NYSE calendar).
- If closed → signal scans and journal entries continue; **no orders may be placed**, including exits. Exception: a hard-stop exit (§0.3) that fires after market close is queued for the next open and logged.

---

## 1. Mode handling (paper → live transition)

The SOP reads `mode.toml` at repo root **on every invocation, before any order tool is touched.** If the file is missing, create it with the paper-default block in §1.1 and report to the user.

### 1.1 `mode.toml` schema

```toml
# /Users/domorand/workspace/personal/ai-trading-agent-sop/mode.toml

# "paper" → SOP runs end-to-end but never invokes a Robinhood MCP order tool.
# "live"  → SOP places real orders.
mode = "paper"

# Empty list = all tickers eligible when in live mode.
# Populated list = ONLY these tickers may receive live orders.
# Use to stage the rollout: ["AAPL"] → ["AAPL","MSFT"] → ["AAPL","MSFT","NVDA"] → [].
live_allowlist = []

# true  = always pause for explicit user "yes" in chat before placing each live order.
# false = auto-execute live orders (still gated by allowlist, risk caps, kill switch).
require_manual_confirm = true

# Optional global blocklist — overrides everything.
block_tickers = []

# Daily loss cap as a fraction of Agentic-account equity. Halts new entries on breach.
daily_loss_cap_pct = 0.02

# Total cluster-signal exposure cap.
max_cluster_exposure_pct = 0.25

# Per-ticker cap.
max_ticker_exposure_pct = 0.05
```

### 1.2 Paper mode behavior (default)

- All signal scans run normally.
- All pre-trade review blocks are emitted.
- **No MCP order tool is invoked.**
- `trade-log.jsonl` entries are appended with `mode: "paper"` and `result: "paper"`.
- Use this mode to build a paper track record before flipping to live.

### 1.3 Live mode behavior

- Pre-flight checks (re-run on every order):
  1. `tools/list` returns Robinhood order tools.
  2. Account ID returned by the MCP matches the user-confirmed **Agentic** account ID. Do not place orders against the primary individual account.
  3. Ticker is in `live_allowlist` (or allowlist is empty).
  4. Ticker is not in `block_tickers`.
  5. Risk caps in §2 not breached.
  6. `KILL_SWITCH` file absent.
  7. `trade-log.jsonl` writable.
- If `require_manual_confirm = true`: emit a one-line confirm prompt and wait for explicit `yes` before placing.

### 1.4 Staged transition recipe

Move along this ladder; do not skip rungs.

1. **Paper for ≥ 30 trading days.** Review `trade-log.jsonl` and `journal/*.jsonl` for: signal noise, missed exits, sizing sanity.
2. **Live + manual-confirm + 1–3 allowlisted tickers.** `mode = "live"`, `require_manual_confirm = true`, `live_allowlist = ["AAPL"]` (or your choice). Run for ≥ 2 weeks. Aim for ≥ 5 live orders to validate the wiring.
3. **Widen allowlist.** Add 2–5 tickers at a time.
4. **Open allowlist.** `live_allowlist = []`. Manual confirm still on.
5. **Auto-execute.** `require_manual_confirm = false`. Only after a written record of stable behavior.

Rollback is always: edit `mode.toml`, save, next invocation honors it.

---

## 2. Watchlist & cash reserve (`watchlist.json`)

A second config file at repo root, alongside `mode.toml`. **Required.** The SOP reads it on every invocation.

### 2.1 `watchlist.json` schema

```json
{
  "watchlist": [
    { "symbol": "SPY",  "description": "S&P 500 ETF — baseline market exposure",   "max_allocation_pct": 15 },
    { "symbol": "QQQ",  "description": "Nasdaq ETF — tech sector exposure",          "max_allocation_pct": 10 },
    { "symbol": "NVDA", "description": "GPU/AI infrastructure — high conviction",   "max_allocation_pct":  8 },
    { "symbol": "AAPL", "description": "Large cap tech — stability anchor",          "max_allocation_pct":  8 },
    { "symbol": "MSFT", "description": "Cloud/enterprise — AI infrastructure play",  "max_allocation_pct":  8 }
  ],
  "cash_reserve_pct": 20
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `watchlist` | yes | Array of permitted symbols. Empty array = no symbol restriction. |
| `watchlist[].symbol` | yes | Uppercase US-listed ticker. |
| `watchlist[].description` | yes | Free text. Surfaced in the journal so future-you remembers the thesis. |
| `watchlist[].max_allocation_pct` | yes | Per-symbol cap as percent of total equity (0–100). Replaces the 5% default in §0.1. |
| `cash_reserve_pct` | yes | Minimum cash to hold, as percent of total equity (0–100). The SOP may never deploy past `(100 − cash_reserve_pct)%`. |

### 2.2 Universe-filter behavior

- **Populated watchlist (≥1 symbol):** no order may be placed for a ticker not in the watchlist. Signals that fire on off-watchlist tickers are still scored and logged in the journal's Market Research section with `Decision: Skipped — not on watchlist`. This is the recommended state for live trading.
- **Empty watchlist (`[]`):** no symbol restriction. Any ticker that survives signal scoring and the decision framework can be traded. Useful for paper-mode discovery.

### 2.3 Cash reserve behavior

- Compute `deployed_pct = (total_equity − cash_balance) / total_equity` on every loop pass.
- A new buy may not push `deployed_pct` above `(100 − cash_reserve_pct) / 100`.
- Worked example: `total_equity = $20,000`, `cash_reserve_pct = 20`. The SOP may deploy up to $16,000 (80%) and must keep ≥ $4,000 in cash. If the candidate buy would push deployed past $16,000, reduce `qty` so it exactly fits — never trim below the §3.1 minimum sizing.
- The reserve floor is checked **after** intraday losses. If a hard-stop exit (§0.3) releases cash, the SOP may redeploy on the next loop pass.

### 2.4 Per-symbol cap interaction with §0.1

The per-symbol cap that applies is, in order of precedence:

1. `watchlist[].max_allocation_pct` for that symbol, if the symbol is in the watchlist.
2. `max_ticker_exposure_pct` from `mode.toml` (default 5%), for any symbol not in the watchlist.

Either way, **some** cap is always enforced. The SOP may never compute "no cap applies."

### 2.5 Watchlist edits

The watchlist is the user's stated universe. The SOP **never** writes to `watchlist.json` programmatically. If a signal surfaces a compelling off-watchlist ticker, log it in the journal and suggest a watchlist edit in the End-of-Day Reflection — the user decides whether to add it.

---

## 3. Position sizing & risk caps

Driven by the conviction tier emitted by `politician-signal.md` or `insider-signal.md`.

### 3.1 Tier → target size

| Tier | Position size (as % of Agentic-account equity) |
| --- | --- |
| skip | 0% (no trade) |
| low | 0.5% |
| medium | 1.0% |
| high | 2.0% |

`qty = floor((equity × tier_pct) / limit_price)`.

Then **clamp downward** to satisfy:

- the per-symbol cap from §2.4 (watchlist value or 5% default), and
- the cash reserve floor from §2.3 (`deployed_pct` may not exceed `1 − cash_reserve_pct / 100`).

If clamping reduces `qty` below 1 share, skip the trade and log `Decision: Skipped — sizing collapsed to zero (reserve or per-symbol cap)`.

### 3.2 Hard caps

- **Per-ticker:** total exposure ≤ the cap derived in §2.4. New buy may not push past.
- **Cluster total:** sum of all open cluster-signal positions ≤ `max_cluster_exposure_pct` (default 25%).
- **Cash reserve floor:** total deployed capital ≤ `(1 − cash_reserve_pct / 100) × total_equity` (§2.3).
- **Daily loss cap:** if realized + unrealized P&L for the day ≤ `−daily_loss_cap_pct × equity` (default −2%), halt new entries for 24h. Existing positions and their exits continue to work.

### 3.3 Liquidity floor

- Skip any ticker with 20-day average daily dollar volume < $5M.

---

## 4. Entry & exit rules

### 4.1 Entry

- **Order type:** limit only. Never market.
- **Limit price:** `last_quote × 1.002` (20 bps slippage budget on top of the ask).
- **Spread filter:** if `(ask − bid) / mid > 0.005` (50 bps), skip — wide spread is a quality flag.
- **Time-in-force:** day order. Re-evaluate next session if not filled.
- **One entry per signal:** do not split into multiple child orders for cluster-signal trades.

### 4.2 Exits (this is how positions close — Signal A / B never produce sells)

Two stops are always in flight. The **wider** one closes the position; whichever fires first wins.

- **Hard stop (§0.3):** −8% from entry. Bright-line, never adjusts.
- **Trailing stop:** starts wide at 12% from peak and **tightens as profit grows** (see §4.3 — Profit-Protection Trailing Stop).

| Exit reason | Trigger | Action |
| --- | --- | --- |
| Hard stop | −8% from entry (§0.3) | close 100% at limit = bid × 0.998, no waiting |
| Trailing stop (profit-protection) | drawdown from peak ≥ tier % (§4.3) | close 100% at limit = bid × 0.998 |
| Time stop | 30 trading days held | close 100% at limit = bid × 0.998 |
| Take profit (partial) | +25% from entry | close 50% |
| Take profit (final) | +50% from entry | close remaining |
| Insider sell kill | CEO or CFO Form 4 `S` on same ticker | close 100% |
| Kill switch | `KILL_SWITCH` file appears | freeze new entries; existing exits continue |

### 4.3 Profit-Protection Trailing Stop (activates after a significant gain)

The trailing stop **tightens** as unrealized profit grows. Track two values per open position:

- `peak_mark` — the highest mark-to-market price seen since entry. Update on every SOP loop pass.
- `gain_pct` — `(peak_mark − entry_price) / entry_price`.

Map `gain_pct` to a trailing band and a hard-stop ratchet:

| Position gain (peak vs entry) | Trailing band (drawdown from `peak_mark`) | Hard-stop ratchet (replaces §0.3 floor) |
| --- | --- | --- |
| < +15% | 12% | −8% from entry (unchanged) |
| +15% to +24% | **7%** | move hard stop to **break-even** (entry price) |
| +25% to +49% | **5%** (partial TP closes 50% here) | move hard stop to **+10% from entry** |
| ≥ +50% | **3%** (final TP closes remainder here) | move hard stop to **+25% from entry** |

**Activation example.** Enter ACME at $40, qty 100.
- Mark climbs to $46 (`gain_pct = +15%`). Trailing band tightens 12% → 7%. Hard stop ratchets from $36.80 to **$40.00** (break-even). A winner can no longer turn into a loser.
- Mark climbs to $50 (`gain_pct = +25%`). Take 50% off (close 50 shares). Trailing band tightens to 5%. Hard stop ratchets to **$44.00**.
- Mark climbs to $60 (`gain_pct = +50%`). Close remaining 50 shares (final TP).
- Alternative path: mark hits $50, then retraces to $47.50 (5% drawdown from peak $50). The 5% trailing band triggers — close the position.

**Implementation rules.**

1. The trailing band and hard-stop ratchet only ever **tighten** — they never widen, even if `peak_mark` later prints lower. (`peak_mark` itself only ever rises.)
2. On every SOP loop pass, for each open position:
   - Re-read the current mark.
   - Update `peak_mark = max(peak_mark, current_mark)`.
   - Recompute `gain_pct`.
   - Look up the trailing band and the hard-stop ratchet in the table above.
   - If `current_mark ≤ peak_mark × (1 − trailing_band)` **or** `current_mark ≤ ratcheted_hard_stop`, submit a closing limit immediately.
3. Persist `peak_mark` and the active tier in the journal so the state survives across SOP runs. Suggested file: `positions.jsonl` (one record per `ticker × entry_intent_id`).
4. The take-profit rows in §4.2 fire *in addition to* the trailing stop. If the market gaps through both, prefer the more conservative (closes more / closes earlier).
5. If the trailing band triggers between regular sessions, queue the close for the next open per §0.5.

### 4.4 Cooldown

After fully exiting a ticker, **10 trading days** before re-entry on a new cluster.

---

## 5. Combined-signal bonus

Strongest single edge in the SOP:

> When **two or more** independent signals fire on the same ticker with their median trigger dates within **±30 calendar days**, upgrade conviction by **one tier** (capped at `high`).

Today's signals are A (politicians) and B (insiders). A draft Signal C (social media) is defined in `references/social-signal.md` but is **not** wired in until the user activates it.

Examples:
- Signal A: medium (score 65). Signal B: low (score 50). Same ticker, within 30 days. Combined: take `max(tier_A, tier_B) = medium`, then upgrade once → **high**.
- Signal A: high. Signal B: skip. No combine — Signal B did not fire.

Record `signal_source: "A+B"` (or `"A+C"`, `"B+C"`, `"A+B+C"` once C is active) in the trade-log entry.

---

## 6. Order placement procedure (step by step)

For each candidate that survives all gates:

1. Compose the pre-trade review block:
   ```
   ticker=ACME  signal=B  score=78  tier=medium  qty=120  limit=42.25  max_loss=$507  account=Agentic-…XYZ
   ```
2. **Append a pending entry to `trade-log.jsonl`** (see §7 schema) with `result: "pending"`.
3. If `mode = "paper"`: rewrite the same line (by re-appending — the JSONL is append-only, so write a `result: "paper"` entry with the same `intent_id`).
4. If `mode = "live"`:
   - If `require_manual_confirm = true`: prompt the user `Place order? [yes/no]`. Wait for `yes`.
   - Invoke the Robinhood MCP order tool (limit, day, equity, buy).
   - On response: append a closing entry with `result: "submitted"` (or `"rejected"` / `"error"`) and the `mcp_response` payload.
5. Update `journal/{YYYY-MM-DD}.md` whether or not the trade went through:
   - Add a row to the **Trades Executed** table for every order placed (paper or live).
   - Add a row to **Positions Closed** for every exit (stop-out, take-profit, trailing-stop).
   - At end of session, fill in **End-of-Day Reflection**.

The trade-log line **must** be written before the MCP order tool is invoked. This is the SOP's core auditability guarantee. The markdown journal is updated alongside but is human-facing.

---

## 7. `trade-log.jsonl` schema

One JSON object per line, append-only, at repo root.

```json
{
  "intent_id": "2026-06-12T14:03:21Z-ACME-B",
  "timestamp_utc": "2026-06-12T14:03:21Z",
  "mode": "live",
  "signal_source": "B",
  "ticker": "ACME",
  "side": "buy",
  "qty": 120,
  "limit_price": 42.25,
  "conviction_tier": "medium",
  "conviction_score": 78,
  "cluster_window_days": 10,
  "cluster_members": ["Jane Doe (CEO)", "John Roe (Director)"],
  "aggregate_amount_usd": 430000,
  "account_id_masked": "…XYZ",
  "mcp_tool_called": "robinhood.place_limit_order",
  "mcp_response": {
    "order_id": "abc-123",
    "status": "queued",
    "raw": "<full response payload>"
  },
  "result": "submitted",
  "error_msg": null,
  "rules_version": "2026-06-12"
}
```

### 7.1 Field semantics

| Field | Required | Notes |
| --- | --- | --- |
| `intent_id` | yes | Unique. Format: `{ISO_TS}-{TICKER}-{signal_source}`. Re-used across the pending/closed pair of entries. |
| `timestamp_utc` | yes | ISO 8601, UTC. |
| `mode` | yes | `"paper"` or `"live"`. |
| `signal_source` | yes | `"A"`, `"B"`, or `"A+B"`. |
| `ticker` | yes | Uppercase US-listed symbol. |
| `side` | yes | `"buy"` for entries; `"sell"` for exits. |
| `qty` | yes | Whole shares (no fractional in V1). |
| `limit_price` | yes | The limit price actually sent to the MCP. |
| `conviction_tier` | yes | `low` / `medium` / `high`. |
| `conviction_score` | yes | 0–100 from the relevant signal file. |
| `cluster_window_days` | yes | The window the cluster fits in. |
| `cluster_members` | yes | Human-readable list (insider names + roles, or member names + chambers). |
| `aggregate_amount_usd` | yes | For Signal A this is the midpoint sum of ranges; for Signal B it's the dollar value of insider buys. |
| `account_id_masked` | yes | Last 4 chars of the Agentic account id. Never log the full id. |
| `mcp_tool_called` | live only | Name of the Robinhood tool actually invoked. |
| `mcp_response` | live only | Full structured response, including the broker order id when available. |
| `result` | yes | `"pending"` (pre-call), `"paper"`, `"submitted"`, `"filled"`, `"rejected"`, `"error"`. |
| `error_msg` | optional | Set when `result` ∈ `{"rejected", "error"}`. |
| `rules_version` | yes | Date stamp of the rules file that produced this decision — lets you reconstruct historical behavior after rule edits. |

### 7.2 Append-only discipline

- Never overwrite. Re-append a new line with the same `intent_id` to record a state transition (e.g. `pending → submitted → filled`).
- Reviewers reconstruct state by reading all lines for an `intent_id` and taking the latest.

### 7.3 `trade-log.jsonl` is the single analytics file

`trade-log.jsonl` is the **one file** that contains every trade the SOP has ever attempted — paper or live, buys and sells, fills and rejects, across every day the SOP has run. Analytics tools should point at this file and nothing else.

**Read pattern.** Stream the JSONL line-by-line, group by `intent_id`, and take the last record per intent — that's the final state of each order. Each buy and each sell carries its own `intent_id`; pair entries with exits by `ticker` and chronological order.

**Quick analytics recipes.**

- **jq — latest state per intent:**
  ```bash
  jq -s 'group_by(.intent_id) | map(max_by(.timestamp_utc))' trade-log.jsonl
  ```
- **jq — total realized P&L on closed sells:**
  ```bash
  jq -s '[.[] | select(.side=="sell" and .result=="filled") | .realized_pnl_usd // 0] | add' trade-log.jsonl
  ```
- **DuckDB (zero-install via brew/pip):**
  ```sql
  CREATE VIEW trades AS SELECT * FROM read_json_auto('trade-log.jsonl');
  -- Win rate
  SELECT
    SUM(CASE WHEN realized_pnl_usd > 0 THEN 1 ELSE 0 END)::DOUBLE
    / COUNT(*) AS win_rate
  FROM trades WHERE side='sell' AND result='filled';
  -- P&L by signal source
  SELECT signal_source, SUM(realized_pnl_usd) AS total_pnl, COUNT(*) AS n
  FROM trades WHERE side='sell' AND result='filled'
  GROUP BY signal_source;
  ```
- **pandas:**
  ```python
  import pandas as pd
  df = pd.read_json("trade-log.jsonl", lines=True)
  closed = df[(df["side"] == "sell") & (df["result"] == "filled")]
  print(closed.groupby("signal_source")["realized_pnl_usd"].agg(["sum", "count", "mean"]))
  ```

**Schema additions for analytics-friendliness.** Sells should include these optional fields (added when the sell fills):

- `entry_intent_id` — the `intent_id` of the matching buy (set by the SOP when emitting the exit), so pair-up is exact.
- `realized_pnl_usd` — `(exit_price − entry_price) × qty − fees`.
- `realized_return_pct` — `(exit_price − entry_price) / entry_price`.
- `holding_period_days` — calendar days between entry fill and exit fill.
- `exit_reason` — one of `hard_stop`, `trailing_stop`, `time_stop`, `take_profit_partial`, `take_profit_final`, `insider_sell_kill`, `manual`.

These are computed locally at exit time — they do not depend on additional MCP calls.

The daily markdown journal (`journal/{YYYY-MM-DD}.md`) is the human-facing read for *one* day; `trade-log.jsonl` is the machine-facing read across *all* days. Both are mandatory; only `trade-log.jsonl` is the analytics source.

---

## 8. Daily journal — markdown format (human-facing)

A new file `journal/{YYYY-MM-DD}.md` is created on the first SOP invocation of each trading day and updated through the day. **Mandatory even on no-trade days (§0.4).**

### 8.1 Template

```markdown
# Trade Journal — {YYYY-MM-DD}

## Portfolio Status
- Cash: ${cash_balance}
- Positions: {ticker} ({qty} shares @ ${avg_entry}), ...
- Total Value: ${total_equity}
- Mode: {paper|live}    Market: {open|closed}

## Market Research
### {TICKER}
- 20-day MA: ${ma20} | 50-day MA: ${ma50} — {bullish|neutral|bearish trend} {commentary}
- News: {one-line headline summary + source}
- Signals: A={tier or "—"}, B={tier or "—"}, combined={tier or "—"}
- Decision: {Entered / Held / Skipped — one-line reason}

(repeat per candidate considered today)

## Trades Executed
| Time  | Symbol | Action | Qty | Price   | Reasoning |
|-------|--------|--------|-----|---------|-----------|
| 10:03 | NVDA   | BUY    | 5   | $847.50 | Signal B medium + 20MA>50MA + cash OK |

(If no trades: write `None today.`)

## Positions Closed
| Time  | Symbol | Reason             | Qty | Entry   | Exit    | P&L    |
|-------|--------|--------------------|-----|---------|---------|--------|
| 14:22 | TSLA   | −8% hard stop §0.3 | 10  | $215.00 | $197.80 | −$172  |

(If none: write `None today.`)

## End-of-Day Reflection
{2–4 sentences. What worked, what didn't, what to watch tomorrow. On no-trade days: why nothing fired and what would change that.}
```

### 8.2 Required sections

Every daily journal must have, in this order:

1. **Portfolio Status** — cash, every open position (ticker + qty + avg entry), total value, mode, market status.
2. **Market Research** — one subsection per candidate ticker considered today (not just the ones traded). Cover 20-day & 50-day MA reading, news headline, signal tiers, decision.
3. **Trades Executed** — markdown table; one row per order intent placed today (paper or live). Use `None today.` if empty.
4. **Positions Closed** — markdown table; one row per exit. Include the reason (rule §, e.g. `−8% hard stop §0.3`, `trailing stop §4.3`, `take-profit final §4.2`). Include realized P&L. Use `None today.` if empty.
5. **End-of-Day Reflection** — 2–4 sentences. Required even on no-trade days; that's where the SOP gets tuned.

### 8.3 Update cadence

- File is created at the first SOP invocation of the day, with sections 1, 2, and stubs for 3, 4, 5.
- After every order or exit, the relevant table is appended in place.
- End-of-Day Reflection is written at the **last** SOP invocation of the day, or by 16:30 ET, whichever comes first.

### 8.4 Relationship to `trade-log.jsonl`

| | `trade-log.jsonl` | `journal/{YYYY-MM-DD}.md` |
| --- | --- | --- |
| Audience | machine (audit, replay, P&L) | human (review, tuning, reflection) |
| Cadence | append every order state transition | one file per day, updated through the day |
| Format | append-only JSONL | markdown |
| Required before MCP order call | **yes** | no (write alongside) |
| Source of truth for fills | yes | no — restate from log |
| Sharable | redact `account_id_masked` first | yes |

Both are mandatory. Neither replaces the other.

---

## 9. Kill switch

Presence of a file named `KILL_SWITCH` at repo root (any contents, any size) means:
- No new entries.
- No new exits triggered by signal logic.
- Existing protective exits (trailing stop, time stop, take profit, insider-sell kill) **still run**.
- Signal scans continue and still write to the journal — they just do not produce orders.

To re-enable: delete the file. Log the event in `journal/{YYYY-MM-DD}.md`.

---

## 10. Stop conditions (SOP-level halts)

Halt the loop entirely (no signal scans, no orders) and report to the user when:

- `mode.toml` is missing or unparseable.
- `watchlist.json` is missing or unparseable.
- The Robinhood MCP `tools/list` fails twice in a row.
- The connected account is **not** the Agentic account.
- `trade-log.jsonl` is unwritable.
- The daily loss cap was breached and < 24h has elapsed.
- US regular session is closed (signal scans still run; ordering halts per §0.5).

---

## 11. Versioning & change control

- This file's effective date is recorded in every trade-log entry via `rules_version`.
- After editing this file, bump `rules_version` to today's date in any new SOP run. This lets you reconstruct which version of the rules produced any given historical trade.
- Never edit `trade-log.jsonl` retroactively. Append corrections as new lines with explanatory notes.
- Daily markdown journals (`journal/*.md`) may be edited the same day (e.g. fix a typo in reflection); never edit a prior day's journal — append an `Addendum` section instead.
