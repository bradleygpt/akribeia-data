# Audit-freeze remediation — e3 diagnosis, granular gate, price unfreeze, frontier fix

**Date:** 2026-07-18. **Repo:** quant-historical (local-only, no remote).
**Commits:** `893cb29` (Phase B, granular gate), `4206e20` (Phase D, price frontier).
**Phase C executed:** price frontier unfrozen, 2026-06-30 → **2026-07-17**.
**Phase A: DECISION STOP — H1 confirmed. e3 untouched, awaiting ruling.**

---

## Phase E (verified first) — NO live exposure

Confirmed by inspection, not memory:

| Check | Result |
|---|---|
| Katalepsis (c78q) live ledger | last entry `as_of 2026-07-01`, `book_type live` |
| Aristeia live ledger | last entry `as_of 2026-07-01`, `book_type live` |
| Rebalances inside the freeze window (7/05–7/18) | **zero** — `next_rebalance: 2026-08-03` |
| `c78q.json` (deployed) | `generated_at: 2026-07-02` — predates the freeze |

Both live books were struck on **2026-07-01** using 6/30 data, which was the correct
point-in-time vintage for a 1st-of-month open execution. The freeze began with the 7/05 audit,
**after** both books were set. No live holding was ever selected off frozen prices.

Worth stating explicitly since it is not obvious: `build_aristeia_strategy.py` **does** read
`daily_panel_CLEAN`, so the exposure was real in principle — it simply never fired, because no
rebalance fell inside the window. This was checked, not assumed.

---

## Phase A — H1 CONFIRMED: the rollback was correct, the rebuild is lossy

The evidence is structural and decisive, not statistical.

### The mechanism
`build_e3_pead.py` extracts earnings dates from the SEC submissions cache:

```python
filings = data.get("filings", {}).get("recent", {})
```

It reads **only** the `recent` block. The SEC caps `recent` at **1,000 filings** and pages
everything older into separate files (`filings.files[]`) that this code never opens.

Measured over a 600-file sample of the submissions cache: **344 (57%) are truncated** — they have
paginated older filings that the extractor cannot see. Concrete examples:

| Ticker | Filings in `recent` | Oldest date still in `recent` | Extra pages ignored |
|---|---|---|---|
| ABT | 1,002 | **2018-05-01** | 2 |
| AMD | 1,000 | **2017-07-05** | 2 |
| AIR | 1,000 | **2013-12-30** | 1 |

Every 10-Q/10-K before those dates is invisible to a rebuild today.

### Why only the audit loses rows
The two invocations differ, and that is the whole story:

| Caller | Invocation | Behaviour |
|---|---|---|
| nightly (`orchestrator_v2.py:437`) | `build_e3_pead --data-root DATA` | **incremental tail-append** (default) — retains events captured when they were still in `recent` |
| weekly audit (`weekly_full_audit.ps1:77`) | `build_e3_pead --data-root $DATA --full` | **full rebuild** — re-extracts from the truncated `recent`, so aged-out events vanish |

`--full`'s own help text: *"Force a full rebuild (default: incremental tail-append)."*

### Verdict
**H1.** The 9,443 rows are real historical PEAD events that a full rebuild **structurally cannot
reproduce**. The audit gate was correctly refusing to destroy them — no-destructive-overwrite
discipline working exactly as designed. H2 is ruled out: the drop is not dedup, not filtering, not
a logic change; it is source-window truncation.

Two corollaries that matter for the ruling:
1. This will **never** self-resolve. Every future `--full` audit will fail the same way, and the
   delta will **grow** as earnings season pushes more filings out of `recent`. The identical 9,443
   on 7/05 and 7/12 reflects the quiet pre-earnings-season lull, not stability.
2. e3's accumulated history is therefore **irreplaceable** — it exists only in the live artifact
   and the snapshots. It cannot be regenerated from source.

### For the ruling (e3 untouched, per instruction)
- **Fix the rebuild** — teach the extractor to read `filings.files[]` pagination, making `--full`
  genuinely reproducible. Cleanest, and restores the audit's meaning for this stream.
- **Or exempt e3 from `--full`** — cheap, but leaves a stream whose full rebuild is known-lossy.

I did not modify e3, the snapshot, or the gate's verdict on it.

---

## Phase B — granular audit gate (commit `893cb29`)

The principle was right; the blast radius was not. One stream at 0.32% drift reverted all ~1,498
streams **plus prices and the panel** — and prices/panel are not even part of the stream
comparison.

`audit_compare.py` now returns **scope** via exit code:

| Exit | Scope | Action |
|---|---|---|
| 0 | PASS | promote everything |
| 1 | **QUARANTINE** | clean streams promote; drifting streams reverted individually, rebuilds set aside for inspection; **prices and panel retained**; run still reports FAILURE naming the held streams |
| 2 | ROLLBACK | widespread drift → revert everything (previous behaviour) |

**Full-rollback threshold — AWAITING YOUR CONFIRMATION:**
`max(10, 1% of streams compared)` → **10** at the current 1,498 streams. Rationale: widespread
drift reads as pipeline/source breakage, where per-stream surgery is the wrong tool; localized
drift is the expected corporate-event case. Under this rule the 7/05 and 7/12 runs (1 stream)
would have **quarantined e3 and promoted the prices** — no freeze.

Quarantine is never a silent pass: the script still throws, the run reports failure, and the
held-back streams are named.

**Verified on real parquet data, all three scopes:** 1-of-12 drifting → exit 1 QUARANTINE with
correct quarantine list; 12-of-12 → exit 2 ROLLBACK; identical dirs → exit 0 PASS. The PS1 parses
under PowerShell 5.1 and is ASCII-clean (per its own header warning about ANSI parsing).

---

## Phase C — price unfreeze EXECUTED

**Condition satisfied:** prices were never among the drifting streams (1 of 1,498 drifted, and it
was e3), and stage4 completed successfully in both failed audits — the 7/12 transcript shows a
clean fetch through 2026-07-10 before the rollback destroyed it. No price drift of any kind.

Ran the audit's own step order: `stage4 --full` → `clean_price_contamination --apply` →
`build_daily_panel` → `axia_recompute_value{,2}` → `reconcile_broad_universe --apply`.

| Metric | Before (frozen) | After |
|---|---|---|
| price frontier **p05** | 2026-06-30 | **2026-07-17** |
| tickers at the frontier date | 1,302 @ 06-30 | 1,305 @ 07-17 |
| **panel max date** | 2026-06-30 | **2026-07-17** |
| panel per-ticker p05 / median | — | 2026-07-17 / 2026-07-17 |
| panel tickers / rows | — | 1,333 / 5,042,386 |

stage4: 1,343 tickers with data, 7,534,659 rows, 21 errors — **all non-destructive**
("fetch failed - kept cached data" / delisted names). The `--full` path falls back to the
per-ticker cache on failure rather than dropping a ticker.

**The 18-day freeze is broken.** Backups retained at `prices_long.parquet.bak_20260718` and
`daily_panel_CLEAN.parquet.bak_20260718`.

### Both dated chips verified — and both behave exactly as ruled

| Ticker | Panel rows | Range | Non-null grade cells | Reading |
|---|---|---|---|---|
| **SKHY** | 6 | 2026-07-10 .. 07-17 | **0** | entered as price-only, curated FPI, zero grades — correct |
| **SPCX** | 24 | 2026-06-12 .. 07-17 | **0** | still price-only, N=3-gated, not yet reporting — correct |

Neither was self-healing; both required the unfreeze, as predicted.

### NEW FINDING — SKHY carries a when-issued bar (needs a ruling)
Phase 3 ruled: *"the SKHYV when-issued window 07-10..07-12 is excluded — only bars >= 07-13
ingested."* The refreshed data violates that:

```
SKHY  2026-07-10  close=168.01  vol=107,674,700   <- when-issued, should be excluded
      2026-07-13  close=152.35  vol= 57,284,300   <- first regular-way bar
```

7/11–7/12 are the weekend, so **7/10 is the only trading day in that window** — one contaminating
bar. The `stage4 --full` universe-wide re-fetch pulled SKHY's complete yfinance history and
reinstated it; the Phase 3 exclusion lives in the original ingestion path, not in stage4.

Materiality is low today (SKHY is price-only and unscored, in no strategy) but the defect is real:
168.01 → 152.35 is a **−9.3%** basis discontinuity that would register as a genuine return in any
momentum or return calculation that later consumes it. That is precisely the contamination class
the exclusion existed to prevent. **Surfaced, not silently patched** — fixing it means touching
ingestion logic, which was outside this session's ruled scope.

### On committing the refresh output
Nothing to commit. `.gitignore:3` is `**/*.parquet` — every parquet is excluded repo-wide, and
`stage4_output/` has **zero** tracked files. The repo norm is code-in-git, data-on-disk; the only
tracked artifacts under `stage5_output/` are the ledger JSONs (untouched this session).

---

## Phase D — price frontier (commit `4206e20`)

Two defects, one incident.

**1. The price layer was never checked at all.** `stage4/prices_long` had no entry in `SOURCES`
and no freshness check anywhere in the codebase. The deeper reason an 18-day freeze survived is
not that a check was fooled — **nothing was looking.** That is the more important half of this
finding.

**2. `max()` cannot express "the universe is stale."** Measured on the frozen file:

| Statistic | Value | Reading |
|---|---|---|
| **max()** | 2026-07-17 | looked perfectly healthy |
| p25 / p50 | 2026-06-30 | the truth |
| **p05** | **2026-06-30** | the truth |
| p01 | 2023-03-16 | delisted names — why not lower than p05 |

New `price_universe` source kind takes **p05 of per-ticker max dates**, tol=**7 sessions**
(prices advance weekly via the audit/advancer, never nightly, so a daily tolerance would
false-alarm constantly).

**Verified:** against the frozen file, p05 lag = **12 sessions > tol 7 → STALE → HARD_STALE**. The
freeze would have fired on the first check after 7/05 instead of running 18 days. Against the
refreshed file it reads FRESH at 2026-07-17, and the checker's overall verdict is FRESH.

**Other `max()` instances** (surveyed, not changed per scope): `frontier_check_v1._max_date_in_parquet`
and `orchestrator_v2.frontier()` both use `max()` over a single date column. Those are **correct as
written** — a per-stream artifact has one date axis, no per-ticker dimension to be masked along.
The masking failure is specific to per-ticker price data. No other instance needs the percentile.

### The lesson, stated plainly
Phase 3's two freshly-fetched tickers (SPCX/SKHY) — added by unrelated work — camouflaged an
18-day freeze from every `max()`-based check. **Without that unrelated change the file would have
read 2026-06-30 and the freeze would have been visible weeks earlier.** A percentile frontier is
the structural fix: it only moves when the broad universe moves, and no handful of fresh names can
carry the headline.

---

## Also done
`LOOKBACK_PAD_DAYS` pad reduction: **not taken.** It is a correctness change (it must be validated
against each Pr factor's longest rolling window) and does not belong as a drive-by in a session
already spanning four phases. It stays with the scheduled loader session.

## Next observable event
The weekly audit fires **Sunday 2026-07-19 04:00** with the granular gate live. Expected: e3
quarantines (named in the failure message, rebuild preserved under
`versions/pre_audit_<stamp>/quarantine_rebuilds/`), every other stream promotes, and **prices stay
current**. That run is the real-world proof of Phase B.

## Open, for Bradley
1. **e3 resolution (decision stop):** fix the extractor to read `filings.files[]` pagination, or
   exempt e3 from `--full`. It will not self-resolve and the delta will grow.
2. **Confirm the full-rollback threshold:** `max(10, 1% of streams)`.
3. **SKHY when-issued bar** (new): re-exclude the 2026-07-10 bar, and decide whether stage4's
   universe-wide re-fetch needs a general when-issued guard.
4. Standing: the Phase E PAT still blocks the credential fix.
