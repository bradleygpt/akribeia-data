# Markets-LLM wiring: nightly self-verification, merge, Thesis Engine live

**Date:** 2026-07-07. **Session:** Claude Code (executor), Bradley (rulings). **Status: SHIPPED.**
Prod is green with the Thesis Engine tab live, the landing sun health-driven, the engine
hardened, JPM grounding wired, and both prod verification directions proven.

---

## 1. Phase -1: nightly self-verification (all four PASS)

The 2026-07-06 22:00 nightly was the first fully organic run of the Phase E chain wired the
previous session. All four checks passed before any merge work began:

| Check | Evidence |
|---|---|
| Task Scheduler | "ML Quant Daily Refresh" LastTaskResult 0 for the 22:00 run |
| Heartbeats | all six in window: stage5 00:44:10, earnings 00:44:11, bake 00:49:21, validate 00:50:14 (5,900+ files, 0 markers, 0 parse errors), data_publish 00:50:22 (version 20260707T045016Z, 7 changed), web_push 00:50:25 |
| Frontier | advanced to 2026-07-06 close (prior-close stamping convention) |
| Publish commits | pro-v2 787a40b96 "data: nightly local bake 2026-07-07T04:50Z", pointer 643c14da1 at akribeia-data@a5947237 (publish commit a594723) |

## 2. Merge rulings (Bradley) and execution

Branch wip/markets-llm merged to main via merge/markets-llm. Safety ref
backup/markets-llm-premerge created first (local only, holds the branch-side
ValueQualityTab and MarketRegimeTab; verified before the merge commit).

| Stop | Ruling | Outcome |
|---|---|---|
| 1: MacroOutlookTab (both-added) | (a) keep main's | branch body discarded; main's newer implementation is a feature superset |
| 2: ValueQualityTab (deleted upstream) | (a) drop + delete file | successor QuantPortfolioTab already covers the changes; merged tree grepped for valqual / ValueQualityTab / the bh standalone id: zero danglers in registry, routes, imports, nav |
| 3: registry.tsx (both-modified) | main's tab list; Thesis inserted before Help | no broader IA restructure |
| 4: JPM wiring proof | approved as shown | see section 4 |
| 5: twin-process | (b) single supervised server | see section 5 |

Re-tokenization (Phase 2): the branch's incoming iframe ThesisEngineTab was replaced wholesale,
so the shipped artifacts (ThesisEngineTab.tsx, src/lib/markets.ts, landing sun changes) were
written token-pure against src/theme.ts. Zero raw hex literals introduced by the merge.

## 3. Thesis Engine tab and landing sun

- **ThesisEngineTab (fetch-based, iframe fully removed):** chat/query UI against the engine
  API. Base URL from VITE_MARKETS_URL, bearer from VITE_MARKETS_TOKEN, both Vercel env only,
  nothing hardcoded in src. Every request carries ngrok-skip-browser-warning and Authorization.
  Mount + interval /health probe (no auth, ~4s timeout). Honest states: offline panel with
  retry (no fake content), loading, empty, GPU-yield 503 and rate-limit 429 surfaced as
  themselves, and responses render only after validation (non-empty, no template-leakage
  markers), the same discipline class as the earnings reviewer.
- **Landing Thesis sun:** hardcoded "in development" label removed. One /health probe on
  landing load: reachable = lit with a live label, unreachable = dimmed with an offline label.
  Never lit while down. No polling storm.

## 4. JPM Guide to the Markets grounding (licensed-private)

- context/private/ gitignored BEFORE creation; proof: `git check-ignore -v` resolves
  gtm_3q26_us.txt to the .gitignore rule (line 16). Never committed, never served, never
  rendered. The extracted text lives only inside engine prompt assembly.
- Extraction: context/extract_gtm.py (pdfplumber, philosophy venv) from the local JPM PDF to
  context/private/gtm_3q26_us.txt, **2,270 lines**, page numbers preserved as section headers.
- Prompt assembly: generation/gtm_context.py selects query-relevant pages (6KB cap) into a
  labeled PRIVATE HOUSE CONTEXT block (house attribution, never citation-numbered). Absent
  file = engine behaves exactly as before.
- Guardrail: any response containing a contiguous span of more than 15 words matching the
  context file is rejected, one paraphrase retry, then refusal (gtm_verbatim_refused). Tested:
  16-word span caught, 15-word span passes.
- Refresh is quarterly and manual by design; nothing scheduled.

## 5. Twin-process ruling and the cutover

**Found topology:** two mutual-keepalive pairs contended for :8765, a philosophy-venv pair and
a system-Python-3.12 pair, each supervisor's port check satisfied by whichever server bound
first, so neither ever exited and both respawned their mates. **Ruling: (b) simplify to a
single supervised server.**

**Launch-vehicle enumeration (before killing anything):** Startup folders (user + common),
HKCU and HKLM Run keys, and all scheduled tasks were enumerated. Exactly ONE launcher exists:
the user Startup .vbs. The system-Python pair had NO launcher anywhere (orphan of a manual
start; it could never survive a reboot). Therefore no launcher deletion was needed.

**The one surviving launcher (reboot survival, verify directly):**
- Path: `C:\Users\bmhar\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\start_markets_supervisor.vbs`
- Command: `"C:\Users\bmhar\code\philosophy-llm\.venv\Scripts\pythonw.exe" "C:\Users\bmhar\code\markets-llm\api\supervise.py"` (hidden, cwd markets-llm)
- A session-only run-once scheduled task used for testing was deleted; the .vbs is the only
  registered launch vehicle.

**Supervisor hardening (api/supervise.py):**
- Fail-loud non-retryable classification (the ERR_NGROK_4018 class): loud log,
  SUPERVISOR_FAILED.json marker, exit 3, no silent respawn.
- Retryable: exponential backoff with a respawn cap (5 per 30 minutes, then non-retryable path).
- Heartbeat every 60s (markets_supervisor_heartbeat.json), budgeted in the quant chain
  watchdog (chain_watchdog.py lines 89-91: missing or lapsed heartbeat and the FAILED marker
  alert at the 07:00/19:00 runs). Engine health is product health.
- Single-instance: yield-to-healthy-incumbent (fresh heartbeat + port answering = newcomer
  exits 0), stale incumbent = take over. Boot path runs ZERO subprocesses (empirical finding:
  any subprocess spawned early under a detached or scheduled-task token killed the interpreter
  silently; the same spawns are safe later, so stale-instance cleanup is deferred ~60s into
  the poll loop). Child spawns now set explicit DEVNULL/PIPE handles: a pythonw-parented child
  has no std handles and server.py's boot print died with OSError 22, one root thread of the
  original twin pathology.

**Cutover result:** both pairs killed, single hardened supervisor launched under the venv,
health/auth/CORS verified (section 6). The engine-down drill and restart also passed.

## 6. Token auth and CORS (server.py)

- All /api/* endpoints require `Authorization: Bearer <token>` (or ?token= for SSE/mobile),
  401 otherwise. GET /health stays open (the sun and tab probes are public-page reads).
- Token file: context/private/api_token.txt (verified gitignored before creation, same
  discipline as the JPM file).
- CORS allowlist: the production dashboard origin and localhost dev ports; allowed origins are
  echoed back, others get no CORS header.
- Verified local AND through the tunnel: /health 200 no-auth; /api/status 401 without Bearer,
  200 with Bearer; Access-Control-Allow-Origin echoes the prod origin; a rogue origin gets no
  CORS header.

## 7. Ship and prod verification (both directions)

- Merge pushed as pro-v2 e5b9fd0ef; Vercel green.
- **Env-var surprise:** VITE_MARKETS_URL and VITE_MARKETS_TOKEN were NOT present in Vercel
  (only the three data API keys existed), so the first prod bundle compiled them empty and the
  tab read not-configured. Disclosed plainly: the executor added both vars to Production via
  the linked Vercel CLI using the values Bradley supplied for this session, then redeployed.
  The live bundle (index-jlEg8JT9.js) now carries both.
- Note on the token: a VITE_ var is baked into the public bundle by design of the
  browser-Bearer architecture. It gates casual access and rate-limiting, not secrecy.
- **Direction 1 (engine up):** /health through the tunnel from the prod origin returns 200
  with the correct CORS echo; authed /api/status 200. Sun lit, tab live.
- **Direction 2 (engine down):** engine stopped; /health fails (sun dims, tab shows the
  offline panel, honest-desk). Restarted via the launcher path; /health recovered to
  `{"ok": true, "yielding": false}`.

## 8. Residuals

- The ~940-review supervised AI regen, the preset sweep, JPM refresh automation, and the
  five-domain IA restructure remain out of scope, as specced.
- backup/markets-llm-premerge stays local (not pushed), holding the dropped branch bodies.
- The supervisor boot saga left diagnostic breadcrumb lines in supervise.log history; the
  shipped code logs normally.
- markets-llm commit f6316fe carries phases 5 and 6; the wip/markets-llm exclusion resumes
  now that the session is over.
- If the ngrok static domain ever changes, update VITE_MARKETS_URL in Vercel and redeploy;
  the src never hardcodes it.
