# BOT TEST PLAN — Touch Probability + P(ITM) + max P/L from now (TEST only)

Independent producer plan. Goal is to break the requirement, not repeat it.
Canonical spec: `/workspace/tsla_bot_data/requirements/REQ_TOUCH_PROBABILITY.md`

Implementation under test: **new** `/workspace/tsla_bot_data/touch_probability.py`
imported only on the TEST overlay path (`attach_touch_fields`).

**Not edited this REQ (steering 2026-08-28 ~14:15 ET):** `publish_prod.py`,
`collect.py`, `slots.py`, `publish_test.py` — a separate worker is landing
REQ 1/2/3 PROD sign-off on those files. TEST wiring is composition:
`finalize_test_envelope` (existing, imported, not patched) then
`attach_touch_fields`, then TEST `write_slot_bundle`. No monkeypatch of
`collect.finalize_test_envelope`.

Frozen / hash-locked (must not be written by this REQ):
`profit_center.py`, `snap_derived.py`, `derived.py`, `launch_snap.py`,
`snap_api.py`, `activity.py`. On-demand `d` is not run.
`launch_snap.py --trigger d` is forbidden. GET-only. Order Limit 0.

This plan is written **before** any implementation `.py`. Hashes in §G are
filled immediately after this file lands, still before `touch_probability.py`.

---

## What would make this requirement wrong

1. Using integer `DTE/365` for 0-DTE → `T=0` → null Touch and null P(ITM)
   while hours remain and IV is valid (the bug this REQ exists to fix).
2. Shipping `2 * p_itm` as production Touch without validating it against
   the first-passage model (it exceeds 100 when p_itm>50 and is not equal
   to GBM-with-r=0 first passage).
3. Treating `shorts[].iv` as a decimal (0.477) when the published field is
   **percent** (47.7) — Touch would be ~100× too small.
4. Substituting HV / ATM / a default vol when IV is missing/invalid.
5. Changing `snap_derived.per_leg_expected_loss` T for DTE>=1 (silently
   moves every stored p_itm / EL / balance_line).
6. Overlaying Touch onto PROD via `publish_prod.py` or the live `d` path.
7. Inventing GREEN/AMBER/RED cutoffs for Touch.
8. Computing max-profit/max-loss from **entry credit** instead of this
   snapshot's marks.
9. Mutating `profit_center` / `balance_line` / H1/H1a/H2 / gate families.
10. Identity strings (`[name]|[acct]|[acct]|strings|[repo-owner]`) inside
    JSON payloads. Orders. Unprefixed `coach/stamps/YYYY-MM-DD/coach_NNN.json`.

---

## Formula source (production Touch)

**Method string:** `gbm_first_passage_r0`

Geometric Brownian motion with **stock drift μ_S = 0** (r=0, q=0), the same
convention as existing P(ITM) in `derived._d2` / `snap_derived.per_leg_expected_loss`
(`d2 = (ln(S/K) − σ²T/2) / (σ√T)` — no +rT term).

Log process: `dX = ν dt + σ dW`, **ν = μ_S − σ²/2 = −σ²/2**.
This ν is documented, not optional: it is the Itô correction that already
sits inside `_d2`.

Let `N` = `derived.N` (same erf). First-passage of arithmetic BM starting at 0:

Upper barrier (calls, K > S), `b = ln(K/S) > 0`:

```
P(max S_t ≥ K) = N((-b + νT)/(σ√T)) + exp(2νb/σ²) N((-b − νT)/(σ√T))
```

With ν = −σ²/2: `exp(2νb/σ²) = S/K`, and the first term **equals** call P(ITM)
`N(d2)`. So Touch = P(ITM) + (S/K) N((−b − νT)/(σ√T)) ≥ P(ITM) for OTM calls.

Lower barrier (puts, K < S), `b = ln(S/K) > 0`:

```
P(min S_t ≤ K) = N((-b − νT)/(σ√T)) + exp(−2νb/σ²) N((-b + νT)/(σ√T))
```

With ν = −σ²/2: `exp(−2νb/σ²) = S/K`, first term equals put P(ITM) `N(−d2)`.

Already at/beyond the strike (call S≥K, put S≤K): Touch = 100.0, status
`ALREADY_TOUCHED`. Does not require IV. P(ITM) still computed separately.

Invalid IV (`None`, `<=0`, non-finite): Touch = null, status `NO_VALID_IV`.
Never HV, never default vol.

Rounding: 1 decimal percent, same as existing `p_itm`.

### Why 2×p_itm is NOT production Touch

For **driftless** log (ν=0), reflection gives Touch = 2 N(−b/(σ√T)) = 2×P(ITM)
on that measure. Our measure is ν=−σ²/2, so:

- `2 * N(d2)` is not the first-passage formula above;
- `2 * p_itm` exceeds 100 whenever p_itm > 50 (already-touched / ITM);
- even OTM, the second term is `(S/K) N(...)` not `N(d2)`.

Test T-2PITM records a numeric grid. Production path has no `2 * p_itm`.

---

## T / expiry convention

```
T = actual_seconds_remaining_until_expiry / SECONDS_PER_YEAR
SECONDS_PER_YEAR = 365 * 86400 = 31_536_000
```

**Why 365 not 365.25:** existing P(ITM) is `T = dte/365`. Touch's year length
must match that calendar so a 1-DTE remaining-seconds T is comparable to
`1/365` near the close, not silently 365.25-scaled.

**Expiry clock:** 16:00 `America/New_York` on `shorts[].expiry` (ISO date).
If `expiry` is missing: `as_of` ET calendar date + integer `dte` days, still
16:00 ET (interpreted fallback; named in results). No other close is
documented in snap_derived / derived.

`as_of` for live/fixture: envelope `generated_at` else `live.as_of`, parsed
aware. Do **not** use the test-runner wall clock for fixture T.

If remaining seconds ≤ 0 and not already touched: Touch = 0.0, status `OK`
(test 7 limit). If as_of/expiry cannot form T and not already touched:
Touch null, status `NO_VALID_TIME` (fail closed, not fabricated).

---

## 0-DTE vs DTE>=1 P(ITM) split

| field | 0-DTE | DTE >= 1 |
|---|---|---|
| `p_itm` on TEST envelope | overlay using remaining-seconds T + existing `N(d2)` / `_d2` (same r=0). A valid 0-DTE short with hours left + valid IV must be **non-null**. | **unchanged** stored value from `snap_derived` (`T=dte/365`). Do not recompute. |
| `p_itm` on snap / PROD / live `d` | untouched (`None` today because T=0) | untouched |
| `expected_loss` | not invented; stays null when the live formula had T=0 | untouched |
| Touch | remaining-seconds T, first-passage | remaining-seconds T, first-passage |

`snap_derived.per_leg_expected_loss` is **not** edited. 0-DTE P(ITM) exists
only as a TEST overlay field on the short dict.

---

## Live-like 0-DTE fixtures (test 12)

Frozen from last TEST envelope `coach_pipeline_v1/staging/test_latest.json`
(generated_at `2026-08-28T13:51:01.958-04:00`, spot **346.62**). There is no
on-disk snap with both 350C and 352.5C; snap_044 has only 352.5C ×50.

| short | qty | IV% | short mark | long | long mark | width |
|---|---|---|---|---|---|---|
| 0 DTE 350C | −50 | 47.7 | 0.145 | 357.5 | 0.015 | 7.5 |
| 0 DTE 352.5C | −50 | 55.4 | 0.045 | 355 | 0.025 | 2.5 |

Expiry 2026-08-28 16:00 ET. Seconds remaining from 13:51:01.958 = **7738.042**.
T = 7738.042 / 31_536_000.

Hand max-P/L (test 13):

- 350C: structure mark = 0.145−0.015 = **0.130**; max profit = 0.130×50×100 = **650**;
  max loss = (7.5−0.130)×50×100 = **36850**
- 352.5C: mark = 0.045−0.025 = **0.020**; max profit = **100**;
  max loss = (2.5−0.020)×50×100 = **12400**

---

## Defined-risk max profit / loss from now

For a short vertical, literal:

```
current_structure_mark = short_leg_mark − long_leg_mark
max_profit_from_now    = current_structure_mark × abs(short qty) × 100
max_loss_from_now      = (width − current_structure_mark) × abs(short qty) × 100
```

Marks from THIS snapshot (`positions[]` POS marks), not entry credit.
Pair = existing `shorts[].long` / `shorts[].width` / `shorts[].qty`
(the pair snap_derived already published). Multi-leg furthest-wing display
is used as-is — we do not invent a new pairing.

Naked / missing width / missing marks: those three fields null (fail closed).
Not a Touch status.

---

## A. Spec tests 1–20

| id | test | expected |
|---|---|---|
| SPEC-1 | Barrier already crossed (call S≥K, put S≤K) | Touch 100.0, `ALREADY_TOUCHED` |
| SPEC-2 | 0-DTE hours remaining + valid IV | non-null Touch and non-null P(ITM) |
| SPEC-3 | Near OTM vs farther, else equal | nearer Touch ≥ farther |
| SPEC-4 | More time, OTM, else equal | longer T Touch ≥ shorter T |
| SPEC-5 | Higher IV, OTM | generally higher Touch |
| SPEC-6 | Call/put synthetic (K=S·u vs K=S/u) | both in (0,100]; consistent; **not** required equal under ν=−σ²/2 (down-barrier ≥ up-barrier). Documented interpretation. |
| SPEC-7 | OTM T→0 | Touch → 0 |
| SPEC-8 | IV None / ≤0 / nan | Touch null, `NO_VALID_IV`, never a number |
| SPEC-9 | Already ITM/touched | Touch 100%; P(ITM) independently N(±d2), may be <100 |
| SPEC-10 | Independent fields | Touch formula ≠ p_itm formula; production path has no `2*p_itm`; both keys present |
| SPEC-11 | Valid OTM Touch ≥ P(ITM) | generally ≥; any exception logged with inputs |
| SPEC-12 | Live-like 0-DTE 350C and 352.5C | both OK, non-null Touch and P(ITM), method `gbm_first_passage_r0` |
| SPEC-13 | max P/L from now vs hand calc | 350C 650 / 36850; 352.5C 100 / 12400 (exact cents) |
| SPEC-14 | qty ×50 vs ×10 | dollar P/L exactly 5× |
| SPEC-15 | Profit Center regression | `profit_center.py` sha256 == pre-hash; stored sample price on disk unchanged |
| SPEC-16 | EL / balance_line regression | `derived.py` + `snap_derived.py` sha256 == pre-hash; stored `balance_spot` unchanged |
| SPEC-17 | non-0-DTE P(ITM) regression | overlay does not change `p_itm` for every dte≥1 short on a fixture |
| SPEC-18 | `d` pipeline | `launch_snap.py` + `snap_api.py` sha256 == pre-hash; this REQ does not call `--trigger d` |
| SPEC-19 | Coach PROD / stamps / DR | this REQ does not edit `publish_prod.py`; does not write unprefixed `coach/stamps/YYYY-MM-DD/coach_NNN.json`. If `publish_prod.py` hash differs from snapshot, treat as **external worker** (WARN + record both hashes), not a silent PASS. |
| SPEC-20 | GET-only | `touch_probability.py` and TEST overlay import no `place_order`/`cancel_order`/`replace_order`; envelope `get_only=true`, `order_limit=0` |

---

## B. Bot-own branches

| id | branch | expected |
|---|---|---|
| BOT-B01 | IV = 0, negative, nan, inf, None | `NO_VALID_IV`, null, no HV |
| BOT-B02 | S == K exactly | `ALREADY_TOUCHED` 100 even if IV invalid |
| BOT-B03 | T=0 OTM | Touch 0.0 OK; P(ITM) null or 0 via `_d2` T<=1e-9 |
| BOT-B04 | `shorts[].iv` percent 47.7 vs decimal 0.477 | attach uses percent/100; mis-reading as decimal would miss fixture Touch by orders of magnitude |
| BOT-B05 | DTE>=1 p_itm byte-identical after attach | no overlay |
| BOT-B06 | 0-DTE p_itm overlay does not write `expected_loss` | stays null |
| BOT-B07 | attach does not mutate `profit_center` / `balance_line` / gate | deep-equal those subtrees |
| BOT-B08 | REQ 2/3 fields survive attach | `derived.gate`, `req_status.req3_materiality` kept if present |
| BOT-B09 | naked short (no width/long) | Touch still published; structure mark / max P/L null |
| BOT-B10 | missing marks | structure fields null, Touch still OK if IV/T valid |
| BOT-B11 | privacy scan of attached envelope | no blocked identity strings |
| BOT-B12 | `2*p_itm` vs first-passage grid | 2×p_itm fails equality and can exceed 100; documented |
| BOT-B13 | `collect.py` / `publish_test.py` / `slots.py` not written by this REQ | sha256 vs snapshot (collect/publish_test may change under the other worker → WARN external) |
| BOT-B14 | TEST stamp path is `coach/test/stamps/...`; PROD prefix refused by `assert_test_path` | no unprefixed write |
| BOT-B15 | input short dicts not mutated in place if caller holds the pre-attach list | attach copies env/derived/shorts |

---

## C. Cache / reuse

| item | reuse | missing |
|---|---|---|
| `derived.N` / `derived._d2` | P(ITM) overlay only | T<=1e-9 or bad IV → null p_itm, not 0 fabricated from HV |
| `shorts[]` pairing, IV%, width, long, qty | as published by snap_derived | missing IV → NO_VALID_IV |
| `positions[]` marks | structure mark | missing → null P/L fields |
| `generated_at` | T clock | missing → `live.as_of`; both missing → NO_VALID_TIME |
| snap_032 / test_latest.json | read from disk each test | missing file fails the runner (no stale in-memory fixture) |
| no module-level memo | pure functions | n/a |

---

## D. Defaults / floor / rounding

- Touch and overlay 0-DTE p_itm: `round(x*100, 1)` percent.
- Structure dollars: `round(..., 2)` then compare exact at cents for the hand fixtures.
- Already-touched: `S >= K` call, `S <= K` put (no extra epsilon beyond float).
- `SECONDS_PER_YEAR = 365 * 86400` (not 365.25, not 252 trading days).
- Expiry clock 16:00 ET.
- Method string exactly `gbm_first_passage_r0`.
- No Touch GREEN/AMBER/RED. Existing p_itm flag RED/AMBER is **not** extended to Touch.

---

## E. Most likely failure mode

**Integer DTE/365 on 0-DTE** so `_d2` returns None (`T<=1e-9`) and both
metrics stay null — exactly today's `derived: null expected_loss excluded
for 2026-08-28 C 350 / C 352.5` warnings. Second: **percent vs decimal IV**.
Third: wiring Touch into `publish_prod.py` / `collect.py` while the other
worker owns those files, colliding with REQ 1/2/3 PROD sign-off.

---

## F. Interpreted vs literal

Literal:
- Fields `touch_probability_pct` / `_method` / `_status` on every OPEN SHORT.
- Keep existing `p_itm` key; do not rename.
- First-passage, not 2×p_itm, unless the grid says they match (they will not).
- Invalid IV → null + `NO_VALID_IV`. Already touched → 100 + `ALREADY_TOUCHED`.
- 0-DTE uses remaining seconds. Defined-risk max P/L from current marks.
- TEST only. GET-only. No new thresholds. test_001–test_024 immutable.
- No unprefixed PROD stamps.

Interpreted (called out):
1. `SECONDS_PER_YEAR = 365*86400` to match `dte/365`.
2. 0-DTE **p_itm overlay on TEST shorts only**; DTE>=1 p_itm left on `dte/365`.
   `snap_derived.py` is not patched, so live `d` still publishes null 0-DTE p_itm.
3. Touch for **all** DTE uses remaining seconds (spec T formula is not 0-DTE-only).
4. `shorts[].iv` is percent.
5. Call/put synthetic equality is **not** required under ν=−σ²/2; we assert
   range + down-barrier ≥ up-barrier.
6. Structure P/L uses published two-leg `(long, width)` even when snap_derived
   displayed the furthest wing of a multi-long.
7. TEST wiring is `attach_touch_fields` after existing finalize, **without
   editing** `collect.py` / `publish_test.py` (steering). A one-shot TEST
   publish from the runner writes stamp / latest / DR.
8. SPEC-19: `publish_prod.py` may change under the other worker; hash drift
   is WARN-external, not claimed as this REQ's regression pass.
9. Extra fail-closed statuses `NO_VALID_TIME` / `NO_VALID_SPOT` / `NO_VALID_STRIKE`
   are allowed by the spec's `...`.
10. ALREADY_TOUCHED is checked **before** IV (100% needs no vol).

---

## G. Non-regression hashes

Captured **after this plan file, before implementation**. Must remain
equal at the end of this REQ for the frozen set:

| file | sha256 | role |
|---|---|---|
| `profit_center.py` | `3b0181df30cab27b1c5a3e5c2e5aa4f85665934e2cd72c06c438985450064916` | SPEC-15 |
| `snap_derived.py` | `74b7ad0c0a81d0a6d055d58c28ebb52efd0457ea86e73b8d028454dbc03bde68` | SPEC-16 / EL |
| `derived.py` | `cabb686fc7a0918138950c28883b885cf13b055878a7ef3765857bf49ff54f73` | SPEC-16 / balance_line / `_d2` |
| `launch_snap.py` | `1585124db62b3ea852be1172eab120cc5631f2ffae64078b8b192e4677422a7c` | SPEC-18 |
| `snap_api.py` | `4e85aa8b936933b12804a57a7fe7b050394c5d6af0695b707dbc10084016b781` | SPEC-18 |
| `activity.py` | `d56ed2784cea36f1c5c538df3d2fbcb2303896c85cdd19436219c0af7f5f9384` | 10-min job frozen |
| `publish_prod.py` | `001840575f295dc8ac4b5f939e99676491c07b75e29aa75a9a946e6924cab4ec` | SPEC-19; external worker may edit |
| `coach_pipeline_v1/collect.py` | `895874bb440362e97c15e3b60b9d78873613902d33ecb3e8fe0a41683b287293` | not edited this REQ; external worker may edit |

Forensics (not a FAIL if the other worker changes them):

| file | sha256 |
|---|---|
| `coach_pipeline_v1/publish_test.py` | `b7d855a1cf096ccc655dc834228ec858933a7855f3ff3237ba71584006b1b323` |
| `coach_pipeline_v1/slots.py` | `30c1ba190873b1ca1ae63b7cb416e67cf63be10dbdd4fa0b13477236e364d051` |

Plan-before-impl sha256 of this file's first write: `a002dfcca5da1c4c71c54923a98450dfe4d445ca09430addc9dc78399e37b4e6`.

---

## H. How to run

```
python3 /workspace/tsla_bot_data/coach_pipeline_v1/run_touch_tests.py
```

Writes `coach_pipeline_v1/BOT_TEST_RESULTS_touch.json`.
After PASS of SPEC-1..18 and 20, publishes current-ET TEST slot:
`coach/test/stamps/YYYY-MM-DD/coach_NNN.json`, matching DR, `coach/test/latest.json`.
Does not overwrite test_001–test_024. Does not PUT unprefixed PROD stamps.
Does not call `launch_snap.py --trigger d`.
