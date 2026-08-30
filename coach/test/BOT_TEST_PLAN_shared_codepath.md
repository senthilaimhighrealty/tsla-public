# BOT TEST PLAN — Shared report codepath (TEST only)

Independent plan. Goal is to break REQUIREMENT_shared_report_codepath.md, not repeat it.
Implementation under test: `/workspace/tsla_bot_data/shared_snapshot.py`
wired by `coach_pipeline_v1/publish_test.py` and this runner.
Live `d` (`launch_snap.py`) and `publish_prod.py` stay frozen until this report PASSes.

Sunday 2026-08-30. GET-only. Order Limit 0. Do not consume prod snap_046.
Do not overwrite test_001–test_024. Do not write unprefixed PROD `coach_NNN`.
Touch pairing is a separate stream — do not regress it; do not promote Touch.

Companion net-market-value field-set file is MISSING. Do not invent
`net_liq` / `cost` / `pl_open` / `pl_day` per-leg fields. Share the CURRENT
coach envelope field set.

## What would make this requirement wrong

1. Two separately maintained envelope builders that happen to match today.
2. Coach TEST stamp and snap-shaped TEST file disagree on content keys
   (other than `generated_at` and path metadata: trigger / slot / output_kind / output_path).
3. Adding a dummy field in the builder requires a second writer change.
4. Slot picker wraps or overwrites existing `snap_045.json` / `snap_047.json`.
5. 5s debounce or PROD_SLOTS=100 change on the `d` path.
6. `publish_prod.py` or `launch_snap.py` edited as part of this TEST landing.
7. Identity strings in JSON payloads (login names, account labels, secret-key names).
8. POSTing order endpoints. Burning snap_046. Overwriting immutable TEST fixtures.

## Documented invariants

| name | value |
|---|---|
| shared builder | `shared_snapshot.build_report` |
| TEST coach writer | `publish_test.py` + `shared_snapshot.coach_writer` |
| TEST snap writer | `shared_snapshot.snap_writer` |
| live `d` | frozen; still `snap_api.fetch_and_build(assign_slot=True)` |
| live 5-min | frozen; still `publish_prod.py` → `collect_live` |
| debounce | 5s, `d` only |
| PROD_SLOTS | 100 |
| next unused on disk | 046 (045 and 047 exist; 046 does not) |
| field set | current coach envelope; companion per-leg fields not invented |
| TEST writes | `coach/test/` only |

## Spec tests (requirement §5)

### SPEC-1 — same field set
Call the shared builder twice on the same payload (coach writer, snap writer).
PASS iff content keys match. Allowed value differences: `generated_at` and
path metadata (`trigger`, `slot`, `output_kind`, `output_path`).
FAIL if positions / orders / derived.gate / req_status / profit_center exist
on only one side.

### SPEC-2 — AST / grep proof one builder
Exactly one `FunctionDef build_report` in `shared_snapshot.py`.
`coach_writer` and `snap_writer` both Call `build_report`.
`publish_test.py` Calls `build_report`.
`publish_prod.py` and `launch_snap.py` do not import `shared_snapshot`.
FAIL if either writer inlines a second envelope implementation.

### SPEC-3 — dummy field appears in both
Set `shared_snapshot.EXTRA_FIELDS["dummy_shared_field"] = "codepath_probe"`.
Rebuild both writers. PASS iff both envelopes contain that key and value.
Clear the hook after. FAIL if only one writer shows it.

### SPEC-4 — slot picker regression
Disk glob of `snap_*.json` + `lowest_unused` skips 045 and 047.
Next unused is 046. Forcing those existing files is refused (they remain
on disk, hashes unchanged). `DEBOUNCE_SEC == 5.0`. `PROD_SLOTS == 100`.
Do not write snap_046. Do not run `launch_snap.py --trigger d`.

## Extra bot tests

### BOT-A — hashes frozen
`launch_snap.py`, `publish_prod.py`, `snap_api.py` sha256 unchanged vs
pre-task snapshot. `publish_test_touch.py` / `touch_probability.py` unchanged.

### BOT-B — companion fields not invented
Built envelopes do not grow top-level or per-leg `net_liq`, `net_market_value`,
`pl_open`, `pl_day`. Note the companion requirement file was not in the tree.

### BOT-C — BEFORE vs AFTER field-set
Record coach vs snap top-level key diff from on-disk prod samples (BEFORE)
and from the two TEST writers (AFTER). AFTER content keys must match.

### BOT-D — privacy
JSON bytes that would be PUT are scanned. FAIL on identity-string hits.
Numbers are never a hit.

### BOT-E — TEST publish
Write `coach/test/stamps/YYYY-MM-DD/coach_NNN.json` (current ET slot) from
the shared builder, plus `coach/test/snap_from_shared.json` from the same
payload. Also copy this plan and results under `coach/test/`.
Refuse PROD stamp paths and test_001–test_024.

### BOT-F — GET-only / no 046
Source of shared builder + TEST publisher has no place/cancel/replace.
After the run: no new `snap_046.json`; `last_snap.json` hash unchanged.

## Companion v2 — Schwab pass-through per-leg fields

Companion file is now `requirements/REQUIREMENT_net_market_value_v2.md`.
Shared builder attaches (TEST only):

| published | Schwab source | notes |
|---|---|---|
| net_liq | marketValue | signed dollars, cents; NEVER qty*mark*100 |
| cost | averagePrice * signed qty * 100 (option) | TOS Cost Basis dollars, not from mark |
| pl_open | longOpenProfitLoss / shortOpenProfitLoss | Gain $ |
| pl_day | currentDayProfitLoss | Day Chng $ |

=POS line (primary `d`/snap target):
`1 P 320 -40 0.055 netliq=-220.40 cost=-312.33 pl_open=91.93 pl_day=100.00`

Same fields on coach envelope `positions[]` from the same builder.
Keep 3-decimal mark. Account AV/PL/CASH unchanged. No bucket_bias this pass.
Live `d` =POS format in snap_api.build_position_lines is unchanged (TEST builder rewrites the block).
