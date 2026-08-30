# BOT TEST PLAN — Publish all manual calcs (TEST only)

Independent plan for REQUIREMENT_publish_all_manual_calcs v2.
Implementation: `shared_snapshot.attach_manual_calcs` gated by
`build_report(..., include_manual_calcs=True)`. Default is False so live
`d` (`snap_api.apply_shared_report`) and coach PROD (`publish_prod.py`)
do not gain these fields until the trader signs off.

Sunday 2026-08-30. GET-only. Order Limit 0.
Do not call `launch_snap.py --trigger d`. Do not PUT PROD `coach/stamps`
or `coach/latest.json`. Do not write snap_048. Do not overwrite
test_001–test_024. Do not import `touch_probability`. Do not copy files off this box.

Fixture (canonical numbers): coach_164 17:39 ET spot 348.19
- local: `coach_pipeline_v1/fixtures/coach_164.json`
- also: `/workspace/tsla_bot_data/snap/snap_046.json` (same book moments earlier)
- GitHub GET (do not PUT over PROD): `coach/stamps/2026-08-30/coach_164.json`

Numbered acceptance tests are fixture-only so they match 17:39 exactly.
A live TEST collect is optional extra and will not match those numbers.

## What would make this requirement wrong

1. Live `d` or coach PROD gaining bucket / bucket_bias / rtr before sign-off
   (`include_manual_calcs` default True, or publish_prod/snap_api passing True).
2. Coach calculating bucket bias from expected-loss (Signal 22) instead of
   net_liq Coach Manual §20.11a.
3. Reconstructing net_liq as qty*mark*100 for bucket_bias.
4. Missing rtr on >45 DTE shorts (only filling derived.shorts 0-45 scope).
5. Silently adding long/width/spread risk for >45 DTE (v1 wing pairing; v2 removed).
6. Crashing when distance_dollar is 0.
7. PUT to coach/latest.json or unprefixed coach/stamps. Writing snap_048.
8. Importing touch_probability.

## Documented invariants

| name | value |
|---|---|
| attach | `shared_snapshot.attach_manual_calcs(env)` |
| gate | `include_manual_calcs=True` (default False) |
| live `d` | frozen; `apply_shared_report` does not pass True |
| live 5-min | frozen; `publish_prod.py` does not pass True |
| TEST publisher | `publish_test.py` passes True |
| bucket | NEAR dte<=1, SHORT dte<=5, MID dte<=30, LONG dte>30 |
| bucket_bias | net_liq put_sum/call_sum/heavy_side/imbalance × 4 |
| heavy_side | CALL if abs(call_sum) > abs(put_sum) else PUT (ties PUT) |
| RTR | mark / distance_dollar, 5 decimal places |
| RTR bands | GREEN >0.008; AMBER 0.003–0.008; RED <0.003 OR mark<0.05 |
| distance_dollar | puts (spot-strike), calls (strike-spot) |
| 100% coverage | positions[] (not derived.shorts 0-45 scope) |
| STK | omit bucket; exclude from bucket_bias |
| TEST writes | `coach/test/` only |

## Spec tests (requirement acceptance)

### ACCEPT-1 — bucket_bias.NEAR matches CALL heavy $740.00
On coach_164: NEAR put=-60.00 call=-800.00 → CALL heavy 740.00.
Also SHORT CALL 1027.50, MID PUT 312.00, LONG PUT 14543.50.

### ACCEPT-2 — 320P RTR
1 DTE 320P mark 0.055, strike 320, spot 348.19:
distance_dollar=28.19, rtr=0.00195, rtr_status=RED.

### ACCEPT-3 — >45 DTE 700C distance_dollar
82 DTE 700C short: distance_dollar = 700-348.19 = 351.81.
Currently missing from derived.shorts 0-45 scope; published on positions[].

### ACCEPT-4 — 100% coverage on positions[]
Every short (qty<0 option) in positions[] has rtr, rtr_status, distance_dollar.
Every option position has bucket. STK may omit bucket.

### ACCEPT-5 — no wing pairing beyond 45 DTE
No long/width/spread risk field added for >45 DTE legs.

## Extra bot tests

### EXTRA-LEGS — table from the spec
360C 1 DTE mark 0.355 distance 11.81 rtr 0.03006 GREEN;
600C 110 DTE mark 1.72 distance 251.81 rtr 0.00683 AMBER;
280P 5 DTE 0.07 / 68.19 = 0.00103 RED;
420C 5 DTE 0.09 / 71.81 = 0.00125 RED.

### BOT-DEFAULT-OFF — live path stays frozen
`inspect.signature(build_report).parameters["include_manual_calcs"].default is False`.
`build_report(fixture)` without the flag does not add bucket_bias or rtr.
AST/grep: publish_prod.py and snap_api.py do not pass include_manual_calcs=True.
launch_snap.py unedited.

### BOT-FAIL-CLOSED — distance_dollar 0
rtr null, rtr_status RED. Mark<0.05 is RED even if rtr would be green.

### BOT-FIELDSET — coach vs snap writers
Same payload, include_manual_calcs=True. Content keys match except
trigger/slot/output_kind/output_path (and generated_at).

### BOT-UNTOUCHED — Signal 22 / gate / PC
balance_line, profit_center, gate keys unchanged by attach_manual_calcs.

### BOT-TOUCH / SNAP — negatives
No touch_probability import in shared_snapshot.py.
snap_048 not written. test_001–test_024 not overwritten.
PROD coach/latest.json sha unchanged by this run.

### BOT-E — TEST publish
Via publish_test.py (coach/test/ only). Predetermined coach_NNN slot for
now-ET, window 04:00–19:55. If after 19:55: fail-closed, publish a TEST
file with a non-slot name under coach/test/ (do not invent a slot).
Paired snap at coach/test/snap_from_shared.json from the same build_report
(trigger d vs coach_5min). Results at coach/test/BOT_TEST_RESULTS_manual_calcs.json.

