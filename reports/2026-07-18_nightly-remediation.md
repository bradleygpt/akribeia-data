# Nightly remediation — pr_cluster timeout, verdict semantics, and a frozen price frontier

**Date:** 2026-07-18. **Repo:** quant-historical (local-only, no remote).
**Commits:** `d62b3ca` (Item 1, timeout), `811b0d0` (Item 3, verdict).
**Item 4: DECISION STOP — a lapse, not a design. Not fixed. Awaiting ruling.**

---

## ITEM 4 (headline) — the price frontier has been frozen at 2026-06-30 for 18 days

It is a **lapse**, and it is the silent-staleness failure class living *inside* the freshness
system itself.

### What actually advances the panel frontier
Nothing in the nightly. The nightly's `ohlcv` step refreshes only the `mlpred_v7_data` streams;
**no nightly step touches `stage4_output` or `stage5_output` at all.** The price history
(`stage4_price_history.py --full`) and the panel (`build_daily_panel.py`) are advanced by exactly
two jobs: the monthly rebalance advancer, and the **Quant Weekly Full Audit** (Sundays 04:00).

### The mechanism
The weekly audit ends with an **AUDIT GATE** that compares every rebuilt stream against a
pre-audit snapshot and **rolls the entire run back** on anomalous drift. It has been failing, and
rolling back, every week since 2026-07-05:

| Audit run | Verdict | Streams compared | Nonzero drop | Outcome |
|---|---|---|---|---|
| 2026-06-26 | PASS | 1495 | 0 | promoted |
| 2026-06-28 | PASS | 1495 | 0 | promoted |
| **2026-07-05** | **FAIL** | 1498 | **e/e3 −9,443 (0.32%)** | **ROLLED BACK** |
| **2026-07-12** | **FAIL** | 1498 | **e/e3 −9,443 (0.32%)** | **ROLLED BACK** |

One stream out of 1,498 — `e/e3.parquet`, the earnings/PEAD stream — loses **exactly 9,443 rows**
on rebuild, both weeks. Identical to the row: deterministic, not random drift.

Each failed audit does real work and then throws it away. The 7/12 transcript shows stage4
successfully re-fetching the full universe through **2026-07-10** (7,533,079 rows, 1,341 tickers) —
and then:

```
!!! AUDIT FAILED - anomalous drift. ROLLING BACK to snapshot 20260712 !!!
WEEKLY FULL AUDIT FAILED: AUDIT GATE FAILED - rolled back to pre-audit snapshot
```

The rollback restores the pre-audit `prices_long.parquet`, discarding the fresh prices along with
the e3 rebuild. The gate is doing its job — it is a blunt instrument being tripped by a real but
tiny anomaly, and it takes the price frontier down with it.

### Why nobody noticed: a 2-ticker mask
`prices_long.parquet`'s headline max date reads **2026-07-17**, which looks perfectly healthy.
The per-ticker distribution does not:

| Per-ticker max date | Tickers |
|---|---|
| **2026-06-30** | **1,302** |
| 2026-06-26 | 8 |
| 2026-06-29 | 4 |
| 2026-07-17 | **2** |
| (other, long-dead) | 27 |

The two names at 7/17 are **SPCX and SKHY**, which my own Phase 3 `reconcile --apply` run fetched
fresh from yfinance on 7/17. Two tickers out of 1,343 were making a frozen file look current to
any check that reads `df.date.max()`. Without the Phase 3 work the file would have read 6/30 and
the freeze would have been visible weeks ago.

### Blast radius
**Affected** (everything gated on `prices_long` / `daily_panel_CLEAN`):
- the daily panel and its four pillar scores — frontier stuck at 6/30;
- all five strategies' price-eligibility gate and the factor sleeves;
- any dashboard surface derived from the panel.

**NOT affected** — and this is the saving grace: the entire `mlpred_v7_data` stream layer is
independent and current to 7/17. The broad ML prediction path, c78q live inference, the regime
HMM, and the nightly publish all read the streams, not stage4/stage5. **The published
predictions are not built on 6/30 prices.**

### Implication for the two dated verification chips — both are BLOCKED
This is the part that changes previously-stated expectations:

- **SKHY panel entry** was said to occur "once the frontier crosses 07-13". The frontier cannot
  cross 07-13 while every audit rolls back. SKHY will **not** appear on any timeline until the
  gate is fixed and one audit promotes successfully.
- **SPCX post-08-06 re-eligibility** has the same dependency. Its companyfacts will populate via
  the Tue/Thu/Sat SEC fetch as designed, but the panel row that makes it observable requires a
  promoted audit.

Neither chip is self-healing. Both are downstream of the Item 4 ruling.

### For the ruling
The proximate fix is the e3 −9,443 discrepancy (why does a rebuild deterministically drop 0.32%
of PEAD rows?). The structural question is separate and arguably more important: **should a
whole-run rollback be triggered by drift in one non-binding stream, and should the price refresh
be coupled to the stream audit at all?** Prices advancing is not conditional on e3 being
byte-stable. Not fixed, not investigated further — read-only per the handoff.

---

## ITEM 1 — pr_cluster timeout (commit `d62b3ca`)

`timeout=3600 -> 7200` at the `step_pr_cluster` call site. 2× the worst observed 2,659s;
deliberately not matched to the siblings' 4h.

### Three timeouts, one root cause — and this was the third
| Step | Timeout | Comment at the call site |
|---|---|---|
| `pr_extras` | 14400 (4h) | *"TEMPORARY (2026-06-16): bumped 3600 -> 14400. pr4/5/6 do a FULL per-ticker OHLCV … ~1,534 tickers, which timed out at 3600. Slicing won't help (I/O, not compute). PROPER FIX QUEUED: make the loader partition-aware -> minutes; revert to 3600 then."* |
| `universe_streams` | 10800 (3h) | *"TEMPORARY (2026-06-24): bumped 3600 -> 10800. Same I/O-bound per-ticker OHLCV … timed out at 3600 every run. FIX QUEUED (same as pr_extras)."* |
| **`pr_cluster`** | **3600 → 7200** | **never raised until now** |

Death was an **explicit orchestrator timeout**, not a crash: eleven clean liveness ticks, then
`[TIMEOUT] pr_cluster exceeded 3600s`. It was killed **mid-Pr3** — Pr1 finished 22:27, Pr2 22:56,
kill at 23:06 — leaving `pr3.parquet` a session stale while Pr4/5/6 (built by the separate
`pr_extras` step, which does not depend on pr_cluster) wrote fresh.

Not a step-change: prior 9 nightlies ran 1670–2659s. The margin was eaten by universe growth
(1,534 → 1,780 tickers).

### Partition-aware loader scoping — the premise of both June comments is STALE
The scheduled "proper fix" is **already ~80% built**, which materially changes the estimate:

- `mlpred_v7/streams/pr/ohlcv_io.py` **already implements** the year-partition skip
  (`{ticker}/{YYYY}/{YYYY}-MM.parquet`, skip whole years before `start_date`, keep one prior year
  as rolling-lookback buffer). It is an **uncommitted working-tree edit** — never committed since
  the initial commit.
- `build_data_quality_stream.py` and `build_e3_pead.py` **already pass `start_date`** — and are
  demonstrably fast (232s and 238s tonight).
- `build_pr_cluster.py` **already runs incrementally**, computing a tail start with
  `LOOKBACK_PAD_DAYS = 420` and passing `start_date` to all three loaders.

So pr_cluster is *not* slow because the loader is history-blind. It is slow because
**LOOKBACK_PAD_DAYS = 420** makes it re-read ~2 calendar years of monthly partitions per ticker —
roughly 24 files × ~1,450 tickers ≈ 35,000 file opens per night — and the per-ticker loop is the
floor. Confirmed by the 7/16 run: `date range 2025-05-22 to 2026-07-16`, exactly the 420-day pad.

**Revised scope** (three candidate levers, none implemented, in cost order):
1. **Reduce `LOOKBACK_PAD_DAYS`** to the longest rolling window actually required (if the true
   need is ≤252d, this is a near-halving for a one-line change). Cheapest; needs a correctness
   check against each Pr factor's longest lookback.
2. **Consolidate partitions** from monthly to per-ticker-per-year, cutting file opens ~12×.
   Touches the cache writer and every reader; the real win, and the real work.
3. **Parallelise the per-ticker loop** (process pool). Independent of 1 and 2.

**Effort:** lever 1 ≈ half a session including verification. Lever 2 ≈ one full session, and it
is the one that reverts all three timeouts to 3600.
**Runtime projection** (measured ~1.3–1.5 s/ticker, near-linear): 1,780 tickers ≈ 2,300–2,700s
today. **At 2,500 tickers ≈ 3,200–3,750s — which clears 3600 but leaves the new 7200 only ~2×
margin.** 7200 is adequate to roughly 2,500 tickers and not much beyond. Universe growth ate the
margin once; on this trajectory it will again, and lever 2 is what stops the pattern.

---

## ITEM 2 — stale-propagation policy (as ruled, recorded)

A failed stream step **does not halt the publish**. One stream one session stale, sitting behind
the binding f1 constraint, does not justify discarding an otherwise good chain. But it must never
publish under an unqualified FRESH — which is Item 3's job. No code change beyond Item 3.

---

## ITEM 3 — verdict semantics (commit `811b0d0`)

### 3.1 DEGRADED on a failed core stream step
`check_frontiers` reads **artifact dates only** and never sees step status. A file that exists but
was not rewritten this session is indistinguishable from a healthy one — no date-based check can
express "a step failed". The downgrade is the only mechanism that can.

Composes with, rather than replaces, the date logic: a genuine `HARD_STALE` still wins, being
strictly worse. Core set = `{ohlcv, pr_cluster, pr_extras, universe_streams, n_events, e_pead,
p_analyst, hmm}`.

**Verified** (5/5, logic exercised in isolation — no real run mutated):

| Case | Base | Result |
|---|---|---|
| tonight: pr_cluster failed, dates fresh | FRESH | **DEGRADED** |
| clean run | FRESH | FRESH |
| hard stale wins over degraded | HARD_STALE | HARD_STALE |
| non-core step failed (enrich) | FRESH | FRESH |
| `skipped` is not `failed` | FRESH | FRESH |

### 3.2 Core artifact set — before / after
The Pr set was incomplete in **two** places, not one. The handoff named `frontier()`; the FRESH
verdict actually comes from `SOURCES` in `frontier_check_v1.py`. Both were missing pr3 and pr5.

| Stream | Before | After | Justification |
|---|---|---|---|
| pr1, pr2, pr4, pr6 | ✅ | ✅ | already present |
| **pr3, pr5** | ❌ | **✅** | consumed by predict; pr3 is the stream that failed and was invisible |
| f1, f6 | ✅ | ✅ | represent the SEC-filing layer (tol=25) |
| f2–f5 | ❌ | ❌ (deliberate) | same builder and cadence as f1/f6 — no added signal, only false-stale surface |
| p1, n1, e3 | ✅ | ✅ | already present |

**Honest limit — this addition would NOT have caught 2026-07-18.** pr3 lagged 1 session against
`tol=2`, so the verdict would still have read FRESH. Verified empirically after the change:
`check_frontiers` still returns FRESH with pr3 at 7/16. It catches the **multi-day** case, where a
repeatedly-failing builder freezes one stream. **Change 3.1 is what catches the single-night
failure.**

**Load-bearing side effect, flagged for awareness:** `frontier()` does not only report —
`min(core)` becomes the **inference date**. Adding pr3/pr5 means a lagging Pr stream now holds
inference back rather than letting it run past a stream that lacks that date. Verified live: the
common frontier now reads **2026-07-16 (binding: pr/pr3)** instead of 7/17. This is correct
behaviour, but it is a behaviour change beyond verdict cosmetics, and it resolves itself once
Item 1 lets pr_cluster finish.

### Frontend: no change required — decision stop 2 is NOT triggered
The daily health JSON is observability-only. It is **not** published to `public/data`, and nothing
in the pro-v2 frontend reads a frontier verdict. There is no freshness pill to update. Note:
`system_status.json` carries a `watchdog` key that is currently `null` — if DEGRADED should ever
be user-visible, that is the natural slot, but it is an option, not a requirement.

---

## Context

- **Task limit: PT5H is in effect — settled empirically.** `Get-ScheduledTask` reports
  `ML Quant Daily Refresh -> PT5H`. The raise was applied. (Siblings: Quant Dashboard Data
  Refresh PT72H, Quant Weekly Full Audit PT9H.)
- **Runtime 2h49m** vs the ~3h13m baseline — *faster only because pr_cluster was cut short*.
  Margin against PT5H ≈ 2h11m; the task limit was never in play. Raising pr_cluster to 7200 has
  ample room.

## Commit hygiene
`ops/daily_refresh_orchestrator_v2.py` carries substantial **pre-existing uncommitted** work that
is not this task's (the v2.2 liveness-tick logic, the predict 3600→7200 bump, and more — HEAD is
well behind the working tree). Only my hunks were committed, staged through the git index via
`hash-object` + `update-index` so the **working tree was never modified** — the reset-HEAD route
would have wiped live nightly code out of the tree, which is exactly the wrong move given the
working-tree bake hazard. Verified after both commits: all pre-existing edits intact. No data
artifacts committed. Left on `master`.

## Open, for Bradley
1. **Item 4 (decision stop):** the e3 −9,443 fix, whether price refresh should be decoupled from
   the stream audit, and any republish/rebake. **Both the SKHY and SPCX verification chips are
   blocked on this.**
2. **Schedule the partition-aware session** from the revised scoping above (lever 2 is the one
   that reverts all three timeouts).
3. Standing: the Phase E PAT remains the only thing blocking that credential fix.
