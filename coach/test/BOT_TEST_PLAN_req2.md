# BOT TEST PLAN — REQ 2 (TEST only)

Independent producer plan for consolidating the 10-min gate into the 5-min
TEST coach envelope. Implementation: `activity_fields.py`, `publish_test.py`,
`collect.py` (spot fail-closed only), `slots.py` (trigger_log path).
`publish_prod.py` stamp naming and envelope schema are UNCHANGED.
On-demand `d` path, activity 10-min job, and live cron are not modified.

This plan covers: (A) spec acceptance 1–7, (B) bot-own branches, (C) cache/reuse,
(D) defaults, (E) most likely failure mode, (F) interpreted vs literal,
(G) non-regression hashes.

## What would make this requirement wrong

1. Publishing an envelope where `derived.underlying_price` is previous_close
   and `live.tsla.last` is the session last (today's `activity_1210.json` bug).
2. Forking a second gate instead of calling `derived.build_gate` /
   `wing_ratios` / `expected_loss_from_shorts` and `activity.py` helpers.
3. Counting the 9 guidance signals in `families_firing` / verdict.
4. Fudging `hard_rules_breached` to 0 when H1a is breached.
5. Changing `publish_prod.py` (clock-floored stamps, no gate on live PROD).
6. Patching `activity.py` / pausing the 10-min job / touching `d`-path files.
7. Overwriting `test_001.json`–`test_024.json` or writing unprefixed
   `coach/stamps/YYYY-MM-DD/coach_NNN.json`.
8. Truncating `trigger_log.json` instead of appending.
9. Mutating `profit_center` / `balance_line` while attaching gate fields.
10. Blocked identity strings in published JSON. Orders. `launch_snap.py --trigger d`.

## A. Spec acceptance tests 1–7

| id | test | expected |
|---|---|---|
| SPEC-1 | H1a vs `derived.shorts` 0–1 DTE in the same envelope | `n_shorts` / `min_distance_pct` / `breached` match recomputation at `derived.underlying_price` |
| SPEC-2 | `derived.underlying_price` vs `live.tsla.last` | identical at 2dp; not previous_close |
| SPEC-3 | Parallel-run diff vs current `activity.json` | `families_firing`/`verdict`/`hard_rules.breached` match **or** disagreement is logged with explanation |
| SPEC-4 | `trigger_log.json` after 3 local successful runs | 3 entries, append-only, first entry intact |
| SPEC-5 | Simulated DR write fail | `files_written` missing the DR path |
| SPEC-6 | `d`-trigger snap path | `launch_snap.py` / `snap_api.py` sha256 unchanged |
| SPEC-7 | `profit_center` / `balance_line` | unchanged by attach (same JSON) |

## B. Bot-own branches

| id | branch | expected |
|---|---|---|
| BOT-B01 | Spot mismatch (`live.tsla.last` mutated) | fail-closed, write NONE of stamp/latest/DR/trigger_log |
| BOT-B02 | Missing activity archive for `signal_history` | `build_signal_history(as_of="2026-07-01")` → sessions=0, 8+9 rows still present, no invented dates |
| BOT-B03 | trigger_log append vs overwrite | 2nd write length=2; entry 0 still first payload |
| BOT-B04 | Partial DR fail | files_written lists only what landed; stamp+latest without `dr/coach_NNN.json` |
| BOT-B05 | Gate vs guidance not double-counted | `families_firing == count(four families)`; not plus `guidance.fires_count`; VOLATILITY absent |
| BOT-B06 | H1a from same shorts as `derived.shorts` | injected 0-DTE short inside 4% → both H1a and shorts-recompute breach |
| BOT-B07 | H1 cap 1500 warn 1000; H1a floor 4% warn 5% | limits on published/fixture gate |
| BOT-B08 | `up_momenta_condition == "== 0"`; trailing_40d `"> 64459"` | strings present; consecutive_losing_days only in guidance |
| BOT-B09 | `hard_rules_breached` equals count of breached rules | not fudged to 0 |
| BOT-B10 | put_test_bytes to `test_001`–`test_024` | refused |
| BOT-B11 | TEST publisher given PROD stamp path | refused |
| BOT-B12 | Hashes of publish_prod / launch_snap / snap_api / activity | unchanged vs REQ 1 capture |
| BOT-B13 | Privacy scan of attached envelope | no blocked identity strings in payload |
| BOT-B14 | Collect/spot fail | no trigger_log entry written (local sim) |

## C. What is cached / reused

| item | reuse | missing/stale |
|---|---|---|
| `derived.build_gate` | live function, explicit `spot=live.tsla.last` | empty price_history → null signals, not invented fires |
| `internal/latest.json` | live TEST only, same session, for txs/counters/history | skipped on fixtures; missing → decisions=[] note |
| close-file hydrate | price_history/vix only; option_chain.underlying_price forced to live last | no close file → history empty |
| `load_prior_track` | read TRACK_CACHE + activity.json | unreadable → start from this collect; **never writes TRACK_CACHE** |
| GitHub trigger_log blob | GET array, append, PUT | missing → `[entry]`; corrupt → tombstone + new entry (does not silently drop) |
| 10-min activity.json | local diff only | missing → compared=false |

## D. Defaults / floor

- Spot compare: round to 2dp (snap_derived stores `round(last, 2)`). Literal "identical" would fail-close every quote with extra ticks.
- day_pnl_track slot: 5-min floor (activity uses 10-min). Uncapped (activity slices 40).
- H1a inside-4% is strict `< 4.0` (same as `h1a_from_positions`).
- trigger_log `generated_at` is the slot_info ET timestamp with offset.

## E. Most likely failure mode

**H1a / wing_ratios disagree with activity.json during parallel-run because activity's `derived.underlying_price` is previous_close** (`derived._hydrate_from_close` + `derived._spot` in `derived.py`). That disagreement is the requirement working. Named so it is not "fixed" by copying activity's stale spot into the coach envelope.

Runner-up: GitHub Contents 409 on `trigger_log.json` (append every 5 min). Same sha-retry as other PUTs.

## F. Interpretation vs literal

Literal:
- Fields listed in REQ 2 §2 inside TEST envelope derived (+ top-level `decisions` / `day_pnl_track` matching activity layout).
- Fail-closed on disagreeing spots. Append-only trigger_log after stamp/latest/DR.
- Reuse `derived.py` / `activity.py` functions. Do not patch activity 10-min job.
- PROD envelope schema unchanged. `d` path unchanged. test_001–024 immutable.

Interpreted (called out):
1. Spot equality at 2 decimal places (see D).
2. `internal/latest.json` same-session txs/counters only on `feed_kind=live`, because `snap_api.fetch_and_build` does not export raw transactions without changing the `d` path (forbidden this REQ).
3. price_history still hydrated from close archives (snap_api does not GET daily bars); after hydrate, quote.last and option_chain.underlying_price are forced to live last so `_spot` cannot revive previous_close.
4. day_pnl_track uses 5-min labels and does not write activity's TRACK_CACHE.
5. Parallel-run diff is local (`coach_pipeline_v1/logs/gate_diff_*.json`), never published to PROD.
6. REQ 2 §3 retirement of the 10-min job is **not** done.

## G. Non-regression hashes

Must be unchanged: `publish_prod.py`, `launch_snap.py`, `snap_api.py`, `activity.py`.
10-min job remains enabled (this REQ does not edit `run_activity.py` / `publish_github.py` / cron).
