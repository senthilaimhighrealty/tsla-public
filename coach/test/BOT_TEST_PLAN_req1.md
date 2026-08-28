# BOT TEST PLAN — REQ 1 (TEST only)

Independent producer plan for predetermined enumerable coach stamp slots.
Implementation under test: `coach_pipeline_v1/slots.py` + `publish_test.py`.
`publish_prod.py` stamp naming is UNCHANGED (clock-floored `YYYY-MM-DD_HHMM.json`).
On-demand `d` path, activity 10-min job, and live cron are not modified.

This plan covers: (A) spec acceptance tests 1–10, (B) every implementation
branch, (C) cache/reuse, (D) floor vs round at exact 5-min boundaries,
(E) most likely failure mode, (F) interpretation vs literal, (G) non-regression
hashes, (H) unpublished future TEST slot → clean 404.

## What would make this requirement wrong

1. Inventing a slot for 03:59 ET or 20:00 ET (must fail-closed / skip).
2. Rounding 12:17:30 ET up to slot 101 instead of flooring to 100.
3. Writing `coach/stamps/YYYY-MM-DD/coach_NNN.json` (PROD, no TEST prefix) during this TEST rollout.
4. Changing `publish_prod.py` stamp logic (cron must keep clock-floored names until sign-off).
5. Overwriting `coach/test/test_001.json` through `test_023.json`.
6. Overwriting last-good on collect fail (must write NONE of stamp / TEST latest / DR stamp).
7. Silent PROD-vs-DR payload disagreement for the same slot (must hard-alert).
8. Touching launch_snap, snap_api, activity.py, snap_044.json, or the 10-min activity job.
9. Identity strings from the blocked list inside published JSON payloads.
10. Submitting broker orders (GET-only, order limit 0).

## Literal slot math

```
NNN = floor((minutes since 04:00 ET) / 5) + 1
```

| clock ET | minutes since 04:00 | floor(/5)+1 | file |
|---|---|---|---|
| 04:00:00 | 0 | 1 | coach_001.json |
| 12:15:00 | 495 | 100 | coach_100.json |
| 19:55:00 | 955 | 192 | coach_192.json |

Window 04:00–19:55 ET → slots 001–192. Zero-pad to 3 digits.
Exact 5-minute boundaries belong to that slot (floor, not round).
Times that compute to NNN < 1 or NNN > 192: do not invent a slot; skip publish.

## TEST paths written (never the unprefixed PROD paths)

- `coach/test/stamps/YYYY-MM-DD/coach_NNN.json`
- `coach/test/latest.json`
- `coach/test/dr/stamps/YYYY-MM-DD/coach_NNN.json`
- `coach/test/stamps/YYYY-MM-DD/manifest.json`

Write order on a successful collect: (1) stamp (2) latest (3) DR stamp.
Collect fail → write none of the three.

## A. Spec acceptance tests 1–10

| id | test | expected |
|---|---|---|
| SPEC-1 | Slot for 12:15 ET | 100 → coach_100.json |
| SPEC-2 | Slot for 04:00 ET | 001 |
| SPEC-3 | Slot for 19:55 ET | 192 |
| SPEC-4 | Fetch an unpublished future TEST slot | clean 404, not empty-200 or garbage |
| SPEC-5 | Fetch a published TEST slot whose URL was enumerable up front | GET 200 after publish; all 192 names knowable without listing |
| SPEC-6 | All three writes for one successful run (PROD paths simulated locally) | identical payload sha256 across stamp, DR stamp, and latest |
| SPEC-7 | Stamp write fails, DR write for the same slot succeeds (local sim) | DR path exists at the matching slot number |
| SPEC-8 | latest.json behavior | TEST latest overwritten with this slot; PROD `coach/latest.json` untouched |
| SPEC-9 | Regression: `d`-trigger snap path | launch_snap.py / snap_api.py / snap_044.json sha256 unchanged |
| SPEC-10 | Regression: existing `dr/market_NNN` / `dr/activity_NNN` 5-slot rotation | activity.py sha256 unchanged; those paths not written |

PROD writes in SPEC-6/7 are local sandbox only. No GitHub PUT to `coach/stamps/` without the TEST prefix.

## B. Implementation branches

| id | branch | expected |
|---|---|---|
| BOT-B01 | 03:59:59 ET | SlotError, skip, no files written |
| BOT-B02 | 20:00:00 ET | SlotError, skip, no files written |
| BOT-B03 | exact 04:00:00 / 04:05:00 / 12:15:00 / 19:55:00 | 001 / 002 / 100 / 192 |
| BOT-B04 | 04:04:59.999 and 12:14:59 | stay on 001 and 099 (floor, not round-up) |
| BOT-B05 | 19:55:01 through 19:59:59 | still 192 (floor). Not a new invented slot. 20:00 is out. |
| BOT-B06 | missing previous stamp (publish 112 with no 111) | 111 absent; manifest has 112 only; 111 would 404 |
| BOT-B07 | GitHub 409 sha race | put_bytes retries with fresh sha, then 200 |
| BOT-B08 | collect fail | write NONE of stamp / latest / DR; last-good stays |
| BOT-B09 | partial write fail (step 1 raises, steps 2–3 proceed) | logged step+slot; DR may still land |
| BOT-B10 | TEST publisher given a PROD stamp path | refused |
| BOT-B11 | overwrite test_001 via put_test_bytes | refused |
| BOT-B12 | same slot republish, bytes unchanged | idempotent sha PUT |
| BOT-B13 | privacy hit in payload | skip, write none |
| BOT-B14 | TEST vs DR sha mismatch | hard_alert set, not silent |

## C. What is cached / reused

| item | reuse | missing/stale behavior |
|---|---|---|
| GitHub blob sha | GET then PUT with sha | 409 → re-GET sha, retry (max 3) |
| last-good TEST latest | left in place on collect fail / out-of-window | never overwritten with empty |
| daily manifest | GET existing, upsert this slot, PUT | missing manifest → start `[]` then append (does not invent prior slots as generated) |
| test_001 fixture | never overwritten | if missing, first-write allowed once |
| on-demand `d` debounce file / snap_manifest used[] | not read, not written | unchanged |

## D. Floor vs round (named)

Floor is literal. Round would be a bug.

| clock | floor NNN (correct) | round-to-nearest-5min would give | why it matters |
|---|---|---|---|
| 04:02:30 | 001 | 002 | first slot stolen |
| 12:17:30 | 100 | 101 | spec example 12:15=100 would be skipped mid-window |
| 19:57:30 | 192 | 193 (invented, out of window) | would publish a slot that must not exist |

Exact boundaries 04:00 / 04:05 / 12:15 / 19:55 belong to 001 / 002 / 100 / 192.

## E. Most likely failure mode

**GitHub Contents 409 on `coach/test/latest.json`.** That path is overwritten every successful TEST run, so a concurrent PUT (or a stale sha from GET-then-PUT) is the highest-frequency write conflict. Mitigation already in `put_bytes`: retry up to 3 times with a fresh sha. Named explicitly so a 409 is treated as expected-retry, not as "publish failed, invent a new slot."

Runner-up: running the publisher at 20:01 ET and expecting slot 192 to still be written. Fail-closed skip is correct; the unpublished URL must 404, not receive a late write.

## F. Interpretation vs literal

Literal:
- Formula, 04:00=001, 12:15=100, 19:55=192, 3-digit pad, floor not round.
- TEST paths as specified. PROD path builders exist in `slots.py` for later import but are not written to GitHub.
- Write order 1-2-3. Collect fail writes none. Individual write fail logs step + slot.
- PROD vs DR byte-identical or hard alert.

Interpreted (called out, not silent):
1. "After 19:55" is implemented as NNN > 192, i.e. >= 20:00 ET. 19:56–19:59 still floor onto 192 (same slot, not a new one). This is the only reading consistent with floor-at-exact-boundary.
2. Manifest is a JSON array of `{slot, time, generated}` for slots actually written so far, not a dense 001–192 array of generated:false placeholders.
3. SPEC-5 "without re-pasting" is a coach-tool conversation property. Producer verifies all 192 names are enumerable from the formula and that a published slot GETs.
4. SPEC-6/7 PROD writes are local sandbox files using `slots.prod_paths()`, never GitHub PUTs to unprefixed `coach/stamps/`.
5. Envelope field `slot` stays `"coach"` (existing schema). Slot number lives in the filename + manifest.

## G. Non-regression hashes (capture before, compare after)

Must be unchanged:
- `launch_snap.py`
- `snap_api.py`
- `publish_prod.py` (entire file, including stamp logic)
- `activity.py`
- existing `snap/snap_044.json`

Also confirm: live cron still `*/5 4-19 * * 1-5` and still runs `publish_prod.py`. This plan does not edit that automation.

## H. Unpublished future TEST slot

Pick a slot after now (e.g. coach_192 if still before 19:55 ET). GET `coach/test/stamps/YYYY-MM-DD/coach_NNN.json`. Expect HTTP 404. Never create that file during this run.
