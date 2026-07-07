# Nightly integration: bulk publish + earnings refresh — completion report

**Date:** 2026-07-06 · **Session:** Claude Code · **Status: COMPLETE — verified end-to-end unattended**

Closes Option B's open condition: the untracked bulk partition now has a **regularly automated
update path** (nightly bake → validate → publish → push), proven by a full production-path run
with no human touch, plus both failure drills.

---

## 1. Precondition: local bake proof (the deferred risk — retired)

| Check | Result |
|---|---|
| `bake.py` end-to-end, mlpred_v7 venv, cwd = source repo | **exit 0, 3 min 51 s** |
| Marker scan + JSON parse over full `web/public/data` | **5,935 files · 0 markers · 0 parse errors** |
| Small-file diff vs committed vintage | 12 files, all expected bake outputs (meta, system_status, freshness_manifest, quant_backtest, regime_timeseries, macro_forecasts, pundits, doppelganger, etf, help, market_static, snapshots) |
| c78q live top-3 | **preserved** — bake explicitly refuses to overwrite with the ETL top-8 |
| blob pointer in freshness_manifest | **survived the bake** (upsert, not rebuild) |
| as_of movement | meta 2026-07-06T18:03Z (CI) → 18:31Z (local); system_status re-stamped; c78q target.as_of 2026-07-01 unchanged (correct — live book) |

No STOP condition met. The unattended-bake risk that deferred Phase E is retired.

## 2. Earnings wiring — the data/AI split

- **Earnings DATA** (ingest): `build_cache.py` (yfinance quarterly stmts, earnings dates,
  surprises) → `fundamentals_cache.json`, refreshed by CI at **10:00 + 23:00 ET** on the source
  repo. The nightly now does a **`git pull --rebase` of the source repo** (Phase E stage
  `earnings_data`) so each night's bake consumes the freshest CI-built caches → `quarterly.json`,
  the earnings panel, surprises. Tonight's proof: cache was **3.4 h old** at bake time.
  `build_earnings_quality.py` (deterministic regex, no LLM) now also runs after every bake.
- **Earnings AI reviews** (generation): `build_earnings_reviews.py` + `earnings_reviewer.py`
  (SEC 8-K Item 2.02 + Ollama qwen2.5) stays **manual and validator-gated**. Why not nightly yet:
  the v2-grounded PROMPT_VERSION bump means every cached review is version-stale, so the first
  "incremental" run is actually a **full ~940-entry regen (hours)** — that must be a supervised
  run. After that one manual regen, nightly incremental (only new filings; cache-hit skip is
  already built in) is a one-line addition to `nightly_publish_v1.STAGE5_BUILDERS`-style wiring
  and every generated review passes `validate_review_text` before it can carry a verdict.
- Parked earnings fixes **landed first**: pro-v2 `d32ea72f4` (grounded api/ai.ts prompt +
  response validation, badge gating, quarterly null→gap chart) — **Vercel green**; source repo
  `0bfe40e` (v2-grounded reviewer + validators + purged cache, 301/1241 invalid reviews removed).
  Pulled pro-v2 dev checkout first; the .gitignore slim made `earnings_reviews.json` a bulk file —
  it ships via this data repo now, not git (served copy = the purged 2,695-key set).

## 3. Phase E wiring (final)

New `ops/nightly_publish_v1.py`, called from `daily_refresh_wrapper_v1._run_and_alert` **inside
the existing .refresh.lock, only on a PASS verdict** — bake is a chain step, no new task, no new
contention class. Chain + measured timings from tonight's run:

| Stage | What | Heartbeat | Tonight |
|---|---|---|---|
| stage5_refresh | advancer C1–C7 builders + aggregates (closes the watchdog's stage5 @2d budget) | `stage5_refresh.json` | 39 s |
| earnings_data | source-repo pull (CI caches in) | `earnings_data.json` | 0.7 s |
| bake | `bake/bake.py` + `build_earnings_quality.py` | `bake.json` | 5 m 22 s |
| validate | **mandatory gate**: marker scan + JSON parse + minimum-file floor; any failure blocks publish AND push | `validate.json` | 56 s (5,935 files) |
| publish | `publish_data_repo.py`: hash-compare → this repo → pointer commit | `data_publish.json` | 12 s (1,523 files) |
| web_push | blob-pointer assert → single commit → push (rebase-retry ×3; public/data conflict → discard + rebake ONCE, never hand-resolve) | `web_push.json` | 3 s |

Every stage fail-loud: first failure stops the line (previous vintage stays live), writes
`<stage>_FAILED.json` (watchdog-caught), and the wrapper emails a PHASE-E FAIL alert.
**Blob-consistency (5cc9dd96 class):** the nightly can never commit a blob-less manifest — the
bake upserts (preserves blob), publish re-injects the fresh pointer, and web_push **asserts**
`blob.base_url + commit_sha` before committing.

**Watchdog additions:** `data_publish.json` @2d (missing/lapsed) + akribeia-data last-push age
@2d (read-only `git log` on the publish clone, alarms only when the last publish reported real
changes). Existing web-repo delivery dead-man (48 h on `public/data` commits) untouched and now
fed nightly. Off-machine: pro-v2's `data-deadman.yml` (meta.json age ≥3d, daily 13:00 UTC) backs
all of this from CI.

## 4. Timing vs PT4H + CI interleaving

Measured tonight (manual 14:40 trigger): core chain **2 h 12 m** (predict 59 m, pr_cluster 39 m)
+ FV history ~33 m + EPS snapshot ~5 m + **Phase E 7.2 m** → **≈ 3 h 13 m wall-clock vs PT4H =
~47 m margin**. That is inside budget but at the threshold, and predict/pr_cluster vary ±20 m.

**Mitigation (recommended): raise the task limit to PT5H.** Run once, elevated PowerShell:

```powershell
$s = (Get-ScheduledTask -TaskName "ML Quant Daily Refresh").Settings
$s.ExecutionTimeLimit = "PT5H"
Set-ScheduledTask -TaskName "ML Quant Daily Refresh" -Settings $s
```

(Zero behavioral change; per-step timeouts inside the orchestrator/Phase E remain the real
guards. 22:00 start stays — settled-close discipline.)

**CI vs nightly — why the interleaving is safe:**
- Source repo CI (10:00/23:00 ET fundamentals; backtest workflows) commits only to the source
  repo → consumed by the nightly's `earnings_data` pull. No product-repo contention.
- pro-v2 CI rebake (11:30 + 00:30 ET) commits the same small git set and **preserves the blob
  pointer verbatim**; the nightly publish's pointer commit is a surgical Contents-API write with
  409 retry. Neither can clobber the other's pointer.
- The one real race: CI's 00:30 ET commit vs the nightly's ~01:15 ET push, same files. Handled
  by design: push rejected → fetch + rebase → on public/data conflict **discard ours, pull clean,
  rebake once (~8 m), republish, push**. Never hand-resolved. Optional de-racing (not applied):
  shift the pro-v2 refresh.yml `30 4 * * *` cron to `0 7 * * *` (02:00 ET).

## 5. Failure drills (both passed)

**(a) Broken publish push:** publish clone remote → unreachable host; ran the real publish.
Failed **loudly** at clone-refresh, **exit 2**, `data_publish_FAILED.json` written (stage +
error), **watchdog flagged `job_failed_flag`**, prod pointer stayed `f1bcd576` throughout.
Restored remote → recovery publish pushed cleanly, FAILED flag self-cleared.

**(b) Corrupted validate input:** scratch dir with a conflict-marker file + truncated JSON →
gate **BLOCKED** on both (markers:1, parse:1), FAILED heartbeat written. The drill also exposed
a real gap, fixed on the spot: an **empty scan now blocks too** (min-file floor 100) — a bake
that writes nothing can no longer read as "validated".

## 6. Unattended production-path run + prod spot-checks

Triggered via `Start-ScheduledTask` (the exact production invocation), watched with zero
intervention: orchestrator (PASS, frontier 2026-07-01 → **2026-07-02**, correct — 7/3 was the
July-4 observed holiday) → FV history → EPS snapshot → all six Phase E stages green → task
**Last Result 0**.

| Prod check | Result |
|---|---|
| Bulk via jsDelivr @`d6bd20a2` | `earnings_reviews.json`, `quarterly.json`, `detail_timeseries/ORCL.json`: **as_of 2026-07-06, sha256 MATCH** vs local |
| Git vintage | web_push commit `4416ce78d` on pro-v2 main; meta.json baked 2026-07-06 — **Vercel: success** |
| Pointer | `c850cf845` "point resolver at akribeia-data@d6bd20a2" (this repo's commit), manifest version `20260706T215317Z`, 5,896 files |
| Earnings panel | served reviews = **purged 2,695-key set**, 56 June-2026 filings, newest 2026-06-17; quarterly ORCL latest 2026-05-31 — current season |
| AsOf badges | both sources fresh: git files via the bake commit, bulk via manifest as_of — dual-source resolver intact |

## 7. Unanticipated findings

1. **web_push ordering bug caught pre-flight:** pull-before-commit refuses on the bake's
   unstaged outputs; reordered to commit-first → push → rebase-on-reject → rebake-once.
2. **Validate empty-scan gap** (drill b): fixed with a minimum-file floor.
3. **Phase E is far cheaper than budgeted** (7.2 m vs ~23 m estimate) — bake 5 m, publish 12 s.
4. The watchdog's stage5 @2d budget pre-dated this wiring (the 7/3 finding assumed a nightly
   rewrite that didn't exist) — `stage5_refresh` closes it.
5. `meta.json` is intentionally absent from the bulk manifest — it is a git-side file; the
   dual-source split is behaving exactly as designed.

**Tonight's 22:00 scheduled run is the first fully-organic nightly** — expect: frontier → 7/6,
all six heartbeats refreshed ~01:10 ET, a new pointer here, and a `data: nightly local bake`
commit on pro-v2. The mlpred_publish 3.6d watchdog alert from the weekend has already cleared.
