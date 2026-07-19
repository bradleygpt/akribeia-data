# Pre-rebuild durability backup — verified, plus a second decaying stream

**Date:** 2026-07-18 (backup stamp 2026-07-19T00:52:58Z UTC).
**Backup path:** `C:\Users\bmhar\.akribeia\backups\20260719T005258Z\`
**Status: ALL MATCH — 18,959 files, 1.38 GB, zero failures. RESTORE.md written, restore tested.**
**No rebuild run. No `--full`, no registry clear, nothing written to live data.**

---

## Headline finding: e3 is not the only one, and n1 is worse

Phase 1 was not a formality. **n1 (8-K events) reads the same truncated source as e3** —
`build_n1_8k_events.py:225` uses `data.get("filings", {}).get("recent", {})`, identical to e3 —
but with a critical difference in how it writes:

| Stream | Nightly mode | Consequence |
|---|---|---|
| **e3** | `--full` absent → **incremental tail-append** | RETAINS history captured before entries aged out |
| **n1** | no `--full` flag, no incremental logic → **full overwrite every night** (`n1_out.to_parquet`) | **CANNOT retain. Degrades a little every single night.** |

e3 at least accumulates. n1 is rebuilt from scratch nightly from a source that is shrinking, so
each run silently discards whatever aged out that day. **Today's n1 is the best copy that will
ever exist**, and last night's is already gone.

### The truncation is measurable in both
Distinct tickers carrying signal, by year — this is not real history, it is the `recent` window
receding:

| Year | n1 tickers | e3 tickers |
|---|---|---|
| 2006 | 35 | 33 |
| 2010 | 125 | 121 |
| 2014 | 402 | 401 |
| 2018 | 839 | 830 |
| 2022 | 1,046 | 1,044 |
| 2026 | 1,091 | 1,096 |

In 2010 only ~11% of the current universe has signal. Corroborated at the source: **57% of a
600-file sample of the submissions cache is truncated** (ABT's `recent` reaches back only to
2018-05-01; AMD's to 2017-07-05).

A note on method: per-ticker filing *rates* look perfectly smooth back to 2010 (3.3–4.6 filings
per 90d) and show no cliff at all. Truncation is invisible in that view and only appears in
*ticker coverage*. The reassuring-looking metric was the wrong lens.

---

## Phase 1 — MUST-BACKUP vs SKIP (for your review)

### MUST-BACKUP (irreplaceable) — all included
| Artifact | Size | Why irreplaceable |
|---|---|---|
| `raw/sec_candidates/cache/submissions` | 1.23 GB | **THE decaying source.** Highest-value item here: backing it up is what makes e3/n1/p1 rebuildable *as of today*, permanently. Derived streams are downstream of this. |
| `streams/e` (e3) | 10.3 MB | Incremental accumulation; `--full` cannot reproduce it. Confirmed H1. |
| `streams/n` (n1, n1_with_metadata) | 23.4 MB | Degrades nightly (above). |
| `streams/p` (p1, p1_sec_fullhist) | 4.8 MB | Form-4/insider from the same `.recent` source; `p1_sec_fullhist` (15,355 tickers, 2006–2025) has **no current rebuilder**. |
| `reports/calibration` (posteriors) | 4.4 MB | 2026-06-17 precedent: a rebuild overwrote signal streams with no backup and made old posteriors permanently unreproducible. Stream versioning still unfixed. |
| `stage5_output/ledgers` | 0.1 MB | Live-money records; included regardless of reproducibility. |
| `stage5_output/_axcCL.npz` | small | Signal cache, same 6/17 failure class. |

### SKIP (genuinely reproducible)
| Artifact | Size | Why safe to skip |
|---|---|---|
| `stage4_output/prices_long` | 180.7 MB | Re-fetchable from yfinance — **proven today**, I rebuilt it end-to-end this afternoon. |
| `streams/pr` (pr1–pr6) | 440.1 MB | Derived from the OHLCV partition cache, itself re-fetchable. |
| `streams/f` (f1–f6) | 18.2 MB | From EDGAR **companyfacts**, which returns full history and is *not* subject to the `recent` truncation. |
| `daily_panel_CLEAN` / panels | ~51 MB | Regenerated deterministically from prices + fundamentals. |
| `streams/m` (macro) | small | Re-fetchable (FRED/yfinance). |

Skipping these avoided ~690 MB and hours of copying with no loss of recoverability. **If you
disagree with any SKIP classification, say so and I will extend the backup — it is additive and a
new timestamped directory costs nothing.**

---

## Phase 2 — the snapshot

`C:\Users\bmhar\.akribeia\backups\20260719T005258Z\`

- **Outside every repo** (asserted programmatically before writing — the script refuses to run if
  the destination resolves inside the repo), so no `--full` run or stray script can reach it.
  Same outside-all-repos discipline as the PAT location.
- **Timestamped and non-destructive:** `mkdir(exist_ok=False)`. A second run creates a new
  directory and can never overwrite a prior backup.
- Relative paths preserved, so restore is a straight copy-back.

## Phase 3 — verification (the point of the exercise)

Every file hashed **on both sides** (SHA-256) and every parquet row-counted on both sides.

```
files=18,959   bytes=1.38 GB   failures=0   VERDICT: ALL MATCH
```

| Artifact | Rows | SHA-256 (head) | Result |
|---|---|---|---|
| `streams/e/e3.parquet` | 2,957,392 | `659833e680` | MATCH |
| `streams/n/n1.parquet` | 2,988,715 | `7265cf55c5` | MATCH |
| `streams/n/n1_with_metadata.parquet` | 2,988,715 | `532a11087e` | MATCH |
| `streams/p/p1.parquet` | 66,211 | `1cca2c4159` | MATCH |
| `streams/p/p1_sec_fullhist.parquet` | 275,123 | `cba13f5a9a` | MATCH |
| `reports/calibration/posteriors_c78q_0p78_h21.parquet` | 275,757 | `f1a6ab9ba3` | MATCH |
| `reports/calibration/posteriors_c78q_NOREV.parquet` | 275,750 | `344b1c6517` | MATCH |
| `reports/calibration/posteriors.parquet` | 240,204 | `6eeca474cc` | MATCH |
| `reports/calibration/calibration_summary*.parquet` (3) | 2,454 / 944 / 879 | — | MATCH |
| `stage5_output/ledgers/*.json` (8) | — | — | MATCH |
| `stage5_output/_axcCL.npz` | — | `4a707dc08c` | MATCH |
| submissions cache | 18,939 files | — | **all MATCH** |

`manifest.json` lists every file with size, hash and row count, so a future restore can be
verified **without the source**.

**Restore tested by checksum** (copied to a scratch location and re-hashed — live data never
overwritten): `e3.parquet`, `n1.parquet`, `c78q_ledger.json` all reproduced their manifest hashes
and row counts exactly. Live `e3` mtime unchanged at 07-17 23:49 throughout.

`RESTORE.md` is in the backup directory: full-restore and single-artifact commands (with the
nightly disabled first), plus a manifest-verification snippet.

---

## Phase 4 — durability of the rebuild run itself

### REQUIRED PRECONDITION: `build_e3_pead.py` writes IN PLACE
```python
e3.to_parquet(out_path, index=False, compression="snappy")   # line 404
```
No temp file, no atomic swap. **A crash or restart mid-write corrupts the live artifact** — and
for e3 that artifact is irreplaceable. This must be fixed before any rebuild.

**The fix is a three-line change with a reference implementation already in this repo**
(`build_data_quality_stream.py:302`):
```python
tmp = out_path.with_suffix(".parquet.tmp")
df.to_parquet(tmp, index=False, compression="snappy")
tmp.replace(out_path)          # atomic on the same volume
```
Same pattern also in `build_perstock_streams.py:91` and `build_ticker_recovery.py:157,173`. So
the codebase already knows how to do this; e3 (and n1) simply never adopted it. Not modified in
this session, per instruction.

### Launch mechanism
Launch the rebuild via **Task Scheduler**, not a detached background shell. Precedent: the prior
prediction run was killed ~1 minute in by an app restart, because a shell-detached child dies
with its parent. I hit the same behaviour twice today — background wrappers reported "exit 0"
while the real work continued or died independently of the wrapper.

### Power / sleep
The existing nightly's settings, which a manual overnight run should mirror:
```
DisallowStartIfOnBatteries = False
StopIfGoingOnBatteries     = False
ExecutionTimeLimit         = PT5H
```
Set an `ExecutionTimeLimit` generous enough for a full e3 rebuild plus margin; the `PT4H`
under-sizing already caused one silent kill in this system's history.

---

## Rebuild authorization checklist

| Precondition | Status |
|---|---|
| Backup created, off working tree, timestamped | ✅ `20260719T005258Z` |
| Verified — hashes + row counts, both sides | ✅ ALL MATCH, 0 failures |
| Manifest enabling source-free verification | ✅ `manifest.json` |
| RESTORE.md present, restore checksum-tested | ✅ 3 files verified to scratch |
| e3 atomic write-to-temp + swap | ❌ **NOT MET — writes in place** |

**The rebuild is not yet authorized.** Four of five preconditions are met; the atomic-write fix is
the outstanding one.

## Open, for Bradley
1. **Review MUST-BACKUP vs SKIP above** — confirm nothing irreplaceable sits in SKIP.
2. **Confirm the atomic-write fix** for `build_e3_pead.py` as the first task of the rebuild
   session (three lines, in-repo reference implementation).
3. **NEW — n1 needs its own decision.** It degrades every night and the pagination fix applies to
   it identically. Options: fix n1's extractor in the same session as e3 (they share the defect),
   or at minimum give n1 the atomic-write + incremental treatment so it stops losing history. It
   is arguably more urgent than e3 because the loss is ongoing rather than latent.
4. Standing: the Phase E PAT still blocks the credential fix.

## Timing note
Tonight's nightly (22:00) will rebuild n1 again and shed a little more history — but **today's
copy is now safely captured**. The weekly audit fires Sunday 04:00 with the granular gate from the
previous session; expected outcome is e3 quarantined, everything else promoted, prices retained.
