# NA-fundamentals remediation — Phase 2 (ADR/FPI + N=3 gate) + Phase 3 (SPCX/SKHY)

**Date:** 2026-07-16. **Repo:** quant-historical (local-only, no remote). **Status: COMPLETE.**
Four commits on `master`: `c79e4e2` (Phase 1 grader), `e64be5c` (Phase 2 gates), `c3d3875`
(FPI price-only retention), `f84021b` (Phase 3 adds).

---

## Why this exists
The Phase 1 audit found the broad ML path ranked null-fundamentals names at the very top (NOK #1,
AAOI #2, LQDA #3) — scored on price streams alone with no fundamental confirmation — and the
sector-relative grader rewarded missing data as top-decile (ACI Profitability 12.0 from all-NaN
inputs). Phase 1 fixed the grader; Phase 2 gates the broad ML path; Phase 3 adds two symbols that
exercise the new gates.

## Phase 1 recap (committed `c79e4e2`, previously reviewed)
`grade_num` used `rank(pct=True, na_option="bottom") + .fillna(1.0)`, sorting NaN to an extreme
(A+/12 for higher-is-better pillars, F/1 for valuation) AND demoting the real top names that NaN
placeholders displaced. Fixed to `na_option="keep"` (NaN absent from the ranking) + drop the
fillna (a fully-missing pillar yields NaN, gated downstream). Verified: the four inflated names
(ACI/AAP/CAVA/UNIT) now NaN; zero symbols get a max pillar from NaN input.

## Phase 2 — ADR/FPI exclusion + N>=3 fundamental-stream gate

### The `adr_score < 0.67` boundary bug (composition shift — results not strictly comparable)
The retired `adr_score` (n_classifiers_foreign / 3) gated eligibility at `< 0.67`. But a 2-of-3
classifier majority = 2/3 = 0.6667, which is **not** `< 0.67`, so the documented "2-of-3 majority"
exclusion **never fired** — only a unanimous 3-of-3 was ever caught. **BABA and TSM (unambiguous
ADRs) were in the scoring path the entire time, contrary to the universe spec.** They leave the
scoring path at commit `e64be5c`, so **universe composition shifts here and pre/post results are
not strictly comparable.** Separately, `fillna(0.0)` read absent names as domestic, hiding seven
IFRS ADRs (NOK/SAP/NVO/TM/SONY/SKHY/SKHYV).

### The replacement: `adr_fpi_flag.py` (hybrid, fail-loud)
Curated override -> `fundamentals_master` form-type backbone (PIT `filing_date <= as_of`;
20-F/40-F/6-K and no 10-K = FPI, 10-K/10-Q = DOMESTIC) -> `adr_flag.filing_form_class` secondary
-> **RAISE** on unknown. No `fillna`, no silent domestic default anywhere. Every resolution records
its provenance.

**The curated list is PERMANENTLY load-bearing.** NOK, SKHY, SAP, NVO, TM, SONY and IFRS foreign
private issuers generally are invisible to **every** filing source in this pipeline — they file
IFRS 20-Fs the us-gaap companyfacts extraction never ingests, so they appear in neither
`fundamentals_master` nor `adr_flag`. The curated list is the **only** mechanism that will ever
classify them. It is not a day-one seed that ages out. This is documented in the module docstring
so no future reader assumes the backbone will absorb these names.

### The SHOP finding (the hybrid is right in BOTH directions)
The form backbone corrected the retired classifier **both ways**: SHOP re-domesticated to 10-K
filing and now correctly resolves DOMESTIC via `backbone_primary`, whereas the old classifier had
it 3-of-3 foreign. Evidence the form-type backbone is authoritative, not just a false-negative fix.

### Real run (exact nightly invocation, exit 0)
```
PHASE-2 GATES: n_fund_streams>=3 dropped 260; ADR/FPI dropped 3; kept 927
  adr resolver provenance: {'backbone_primary': 918, 'backbone_secondary': 9}
```
- **Zero fail-louds.** The gate order is load-bearing (N>=3 stream gate FIRST, strict ADR
  resolution only on survivors) — every unresolvable name has < 3 streams, so it is dropped
  before strict resolution is attempted. A comment at the call site explains why reordering
  would raise on ~33 names and kill the nightly.
- **NOK/AAOI/LQDA eliminated** from the broad ML output. New top-25 is entirely >=3-stream,
  non-FPI names (POWL, VIAV, VSAT, MXL, INTC, RKLB, TER, WDC, COHR, MU, ...).
- **Exclusion accounting:** N=3 dropped 260, ADR dropped 3 survivors (DOX/ICLR/PDD/PXS class),
  kept 927. The other ~11 in-universe FPIs fall to the N-gate first (overlap).

### Decision Stop 2 (ruled ACCEPT): net `is_eligible` flip = 7, not 156
Removing `adr_score` from `is_eligible` flipped **7** names to tradeable (HSBC, ASML, SPOT, CNQ,
SU, CLS, ICLR) — well-known 20-F/40-F foreign filers the buggy gate wrongly blocked from
tradeability, now tradeable-but-unscored as ruled; min ADV $140M, zero junk. The gross-156 was
stale tickers + ADV/mcap/age failures, not a universe shift.

### Ruling 2: every FPI retained as a price-only row (commit `c3d3875`)
`build_daily_panel` drops FPIs from the scored set; reconcile previously re-added only names with a
c78q posterior, so ICLR vanished from the panel entirely while ARM survived by accident. reconcile
now sweeps the whole price universe and re-adds any FPI absent from the panel (strict=False
coverage sweep, never raises). Verified: ICLR reappears price-only; 12 FPIs retained; zero FPIs
carry panel grades.

## Phase 3 — SPCX + SKHY (commit `f84021b`)
- **SPCX** (Nasdaq, CIK 1181412, IPO 2026-06-12, domestic, zero 10-Q until ~2026-08-06): prices
  ingested from 06-12; added to `universe_final` with its CIK. Under the N=3 gate it is
  **ineligible for the broad ML path** (0 fundamental streams) — confirmed absent from the
  prediction CSV, gated by N=3 not ADR. Retained as a **price-only panel row** via a new
  reconcile rule: a registered domestic filer that has NEVER reported (`total_10k_or_10q == 0`)
  is kept price-only until its companyfacts populate. **Auto re-check mechanism:** the Tue/Thu/Sat
  `SEC Companyfacts Fetch` job (`fetch_companyfacts.py`) fetches clean-universe CIKs from the SEC
  submissions cache; once SPCX files its 10-Q, its companyfacts populate, f-streams build, and the
  nightly panel/prediction rebuild re-evaluates SPCX against the N=3 gate automatically. No manual
  flag.
- **SKHY** (Nasdaq ADR, regular-way from 2026-07-13; the SKHYV when-issued window 07-10..07-12 is
  excluded — only bars >= 07-13 ingested): prices ingested, added to `universe_final`, and it is
  in the **curated FPI list** (resolves FPI via `curated`). Under Ruling 2 it persists as a
  price-only row permanently. It does not appear in today's panel only because the panel frontier
  is 2026-06-30 (SKHY did not exist then); it enters once the frontier crosses 07-13.
- **ADR ratio guard (mandatory):** `ADR_RATIO = {SKHY: 10, SKHYV: 10}` (1 ADR = 1/10 Korean
  common share) + `require_ratio_applied()`, which RAISES if per-share fundamentals are ever
  attached to a ratio'd ADR without the ratio applied — same failure class as the split-adjusted
  lookahead (price and share-count on different bases). Tested: raises on fundamentals-without-ratio,
  passes when ratio applied or price-only. Prevention, not later audit.
- **Surfaced, not masked:** six already-reporting names (MC, PLNT, STZ, TR, VGNT, VSNT) have prices
  but no panel row — a fundamentals-coverage issue, NOT new IPOs. reconcile logs them and does
  **not** silently add them as price-only. Worth a separate look.
- Paper/informational only; no ledger or live book touched.

## NEW BACKLOG ITEM (logged, not fixed): working-tree bake hazard
A "commit withheld at a decision stop" did **not** withhold behavior: the 22:00 nightly bakes from
the WORKING TREE, so the gated wiring went live into production artifacts before the ruling landed.
Harmless this time (ruling = accept), but a decision stop that does not gate behavior is theater.
Two candidate fixes for a later ruling, neither implemented:
(i) **structural** — the nightly bakes only committed state (stash-or-fail discipline at bake start);
(ii) **procedural** — decision-stop-pending changes never sit in the working tree past 21:00.

## Commit-hygiene note
`predict_returns.py` carried two pre-existing uncommitted hunks (not this work). Only the Phase-2
gate hunk was staged (surgical patch via reset-HEAD + re-insert + restore); the pre-existing hunks
remain untouched in the working tree. Data artifacts (`prices_long.parquet`, `universe_final.csv`,
regenerated panels) were deliberately **not** committed, per the constraint.
