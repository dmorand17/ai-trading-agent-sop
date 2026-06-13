# Risk Review (Two-Agent Pattern)

When `require_risk_review = true` in `mode.toml`, the Trader (main session running SKILL.md) **must** spawn the `risk-reviewer` subagent on every candidate trade and only invoke the Robinhood MCP order tool after an `approve` decision.

The Reviewer's definition lives at `.claude/agents/risk-reviewer.md`. This file documents the Trader-side workflow, the proposal/decision artifacts, and the failure modes.

## 1. Why split

A single agent that both scores a signal and places the order tends to confirm its own thesis. Two failure modes get caught by a second agent that didn't propose the trade:

1. **Stale rule application.** The Trader was last told the cap was 5%; the user bumped a symbol to 8% in `watchlist.json` an hour ago. The Reviewer reads the file fresh.
2. **Bias under conviction.** A high-score signal makes the Trader rationalize bending the 5-question framework. The Reviewer's prompt forces adversarial rule-checking.

The Reviewer **does not re-score the signal.** It only checks rules. This keeps the runtime cost low and the failure mode (Reviewer disagrees with Trader's conviction) impossible.

## 2. Flow

```
Trader (main session, SKILL.md)
   │
   │ scores Signal A / B, applies Decision Framework
   │
   ├─► writes proposals/{intent_id}.json
   │
   ├─► spawns risk-reviewer subagent with the proposal
   │
   │     Reviewer reads:
   │       mode.toml, watchlist.json, rules.md,
   │       trade-log.jsonl, KILL_SWITCH, journal/today.md
   │     Reviewer returns: {decision: approve|reject, reasons: [...]}
   │
   ├─► writes reviews/{intent_id}.json
   │
   ├─► if decision == approve:
   │       append pending trade-log line
   │       invoke Robinhood MCP order tool
   │       append final trade-log line
   │
   └─► if decision == reject:
           append trade-log line with
             result: "rejected_by_risk_review"
             rejection_reasons: [...]
           journal entry: "Skipped — Reviewer rejected: <reasons>"
```

## 3. Proposal artifact

The Trader writes one file per candidate to `proposals/{intent_id}.json`. Schema (canonical fields shown in `.claude/agents/risk-reviewer.md`):

- `intent_id`, `ticker`, `side`, `qty`, `limit_price`
- `signal_source`, `conviction_tier`, `conviction_score`, `cluster_members`, `aggregate_amount_usd`
- `decision_framework` with Q1–Q5 answers
- `account_snapshot` with `account_id_masked`, `total_equity_usd`, `cash_usd`, `deployed_pct`
- `market_status`

`proposals/` is gitignored. The artifact is durable enough to support a two-routine cloud pattern (Routine A writes proposals; Routine B fires when proposals appear, runs the Reviewer, writes reviews).

## 4. Review artifact

The Reviewer (or the Trader on the Reviewer's behalf) writes `reviews/{intent_id}.json`:

```json
{
  "intent_id": "...",
  "decision": "approve",
  "reasons": [],
  "checks_passed": ["A.1", ...],
  "checks_skipped": ["D.4 — liquidity not verifiable from proposal"],
  "reviewer_version": "2026-06-12",
  "reviewed_at_utc": "2026-06-12T14:03:35Z"
}
```

`reviews/` is gitignored. Reviews are immutable once written.

## 5. Trade-log integration

When the Reviewer rejects, the Trader still writes one line to `trade-log.jsonl`:

```json
{
  "intent_id": "...",
  "timestamp_utc": "...",
  "mode": "live",
  "signal_source": "B",
  "ticker": "ACME",
  "side": "buy",
  "qty": 120,
  "limit_price": 42.25,
  "conviction_tier": "medium",
  "conviction_score": 78,
  "result": "rejected_by_risk_review",
  "rejection_reasons": [
    "B.1 position cap: post-trade exposure 6.3% exceeds per-symbol cap 5%"
  ],
  "rules_version": "2026-06-12"
}
```

Add `"rejected_by_risk_review"` to the `result` enum in `rules.md` §7.1 (Field semantics).

## 6. When to require review

In `mode.toml`:

```toml
require_risk_review = true   # default true once feature is enabled
```

- **`true` (recommended for live):** every candidate goes through the Reviewer regardless of mode.
- **`false`:** Trader skips the Reviewer entirely. Use only during the very first paper-mode shakedown when the Reviewer scaffolding itself hasn't been validated.

There is no per-mode override — paper trades should also be reviewed, so the daily journal reflects what would have been blocked in live.

## 7. Trader-side instructions (paste into the prompt when spawning)

The Trader should spawn the Reviewer with this prompt template (substitute the JSON inline or pass the path):

```
You are the risk-reviewer subagent. Review the following trade proposal
against rules.md §0, watchlist.json, and trade-log.jsonl. Return a single
JSON decision object per .claude/agents/risk-reviewer.md.

Proposal:
{INLINE_PROPOSAL_JSON or path to proposals/{intent_id}.json}

Read mode.toml, watchlist.json, references/rules.md, trade-log.jsonl,
and check for KILL_SWITCH. Use the checklist in your system prompt.
No prose, JSON only.
```

## 8. Failure modes & escalations

- **Reviewer times out / returns malformed JSON.** Treat as reject. Log `result: "rejected_by_risk_review"`, `rejection_reasons: ["reviewer_malformed_response"]`. Surface to the user at end of loop.
- **Reviewer rejects 100% of candidates over N consecutive runs.** Likely a rules drift (e.g. cash reserve too high relative to current equity). Surface to the user in the End-of-Day Reflection with a tuning suggestion.
- **Trader bypasses the Reviewer.** Should be impossible in normal operation (the SOP loop in SKILL.md §4 step 5 calls the Reviewer before any order tool). If a trade-log line appears with `result: "submitted"` and there is no matching `reviews/{intent_id}.json`, that's a critical audit failure — investigate immediately.

## 9. Routine-chain variant (optional, advanced)

For the cloud-routine deployment, you can decouple Trader and Reviewer into two routines:

- **Routine A — Trader.** Schedule: 9:35 ET. Writes proposals/, opens a draft PR with the proposals included.
- **Routine B — Reviewer.** Trigger: GitHub `pull_request.opened` on the proposals PR. Reads proposals, writes reviews to a separate branch, merges if all approved.
- **Routine C — Executor (optional).** Trigger: merge of the proposals PR. Reads reviews, executes approved trades.

This is overkill for v1 but documented for when audit pressure increases.

## 10. Validation

Before flipping `require_risk_review = true` in live mode, run for ≥ 30 paper-trading days with the Reviewer active. Confirm:

- The Reviewer approves trades the Trader would have placed correctly.
- The Reviewer rejects trades for cited rule reasons, not vibes.
- `proposals/` and `reviews/` artifacts pair 1:1 in the journal.
- No `submitted` trade-log lines lack a matching `approve` review.
