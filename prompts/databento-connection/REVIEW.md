# Review record — Databento connection prompt

This file documents how `PROMPT.md` was produced: research grounding, the first draft,
the adversarial review findings, and what changed in the re-draft.

## Method

1. **Research** (4 parallel investigations, 2026-07-12):
   - `FedericoEtche/databento-python` clone (v0.78.0, CHANGELOG 2026-05-12) — client APIs, auth, exceptions, live session mechanics.
   - `FedericoEtche/dbn` clone (v0.62.0, CHANGELOG 2026-07-07) — schemas, enums, live `ErrorMsg` codes.
   - databento.com pricing/plan pages and the public `service_offerings` API — what the two **Standard** plans actually include.
   - Databento docs — endpoints, CRAM auth, connection limits, firewall requirements, OPRA/equities gotchas, error semantics.
2. **Draft v1** written from the research digest only (no unverified claims).
3. **Adversarial review** by four independent lenses: technical accuracy, prompt engineering, completeness, usability & security. 44 findings (3 critical, 25 major, 16 minor).
4. **Re-draft (v2 = `PROMPT.md`)** incorporating all findings, then a final two-agent verification pass (accuracy re-check against the repos + internal-coherence simulation).

## Key research facts the prompt is built on

- Plans: "Standard Databento US Equities" and "Standard OPRA", $199/mo each (mid-2026). Equities Standard: historical access to the ~20-dataset US equities bundle (windows: full history for ohlcv/definition/statistics/status, 12 mo for top-of-book, 1 mo for depth) but **live = `EQUS.MINI` + `IEXG.TOPS` only**. OPRA Standard: `OPRA.PILLAR` historical (12-mo window for tick schemas, full for aggregates) and live in **all** schemas; personal/non-professional use, ≤2 devices.
- `OPRA.PINNACLE` no longer exists — the dataset is `OPRA.PILLAR`; `mbp-1`/`tbbo` were removed from OPRA on 2025-05-27 (replaced by `cmbp-1`/`tcbbo`).
- Auth: 32-char keys starting `db-`; all clients fall back to `DATABENTO_API_KEY`. Historical = HTTPS `hist.databento.com`; live = raw TCP `<dataset>.lsg.databento.com:13000`, CRAM, egress TCP 13000–13050 to 209.127.152.0/21.
- Out-of-plan historical requests **do not error** with pay-as-you-go enabled — they bill per-GB; metadata calls (incl. `get_cost`) are free; the portal monthly limit is the backstop.
- Live `ErrorMsg` codes: 1 AuthFailed, 2 ApiKeyDeactivated, 3 ConnectionLimitExceeded (10 sessions/dataset/team on Standard), 4 SymbolResolutionFailed (non-fatal), 5 InvalidSubscription, 7 slow-reader skip.
- Intraday replay (`start=0`, last 24 h) must be requested on the **first** subscribe of a fresh session — it cannot be added to a started session.

## Review findings (condensed) and how the re-draft addressed them

### Critical

| Finding | Fix in v2 |
|---|---|
| Intake asks for prior tracebacks — the most likely artifact to contain a hardcoded key — with no redaction protocol at the moment of the ask (2 lenses flagged independently). | Redaction warning is now mandatory verbatim text appended to **every** paste request; assistant must scan every paste for `db-…`/Basic-auth blobs; explicit rotation → re-verify procedure; ban on `curl -v`/`set -x`/history. |
| The bash-only `curl -u "$DATABENTO_API_KEY:"` cross-check breaks on Windows (cmd doesn't expand `$VAR`; PowerShell aliases `curl`) and invites inlining the literal key. | Per-OS command policy in the interaction contract; curl check labeled macOS/Linux with a separate `curl.exe … $($env:…)` PowerShell form; standing rule to never inline the key into commands. |

### Major (selected; all were applied)

- **Replay instruction was wrong**: "re-subscribe with `start=0`" fails on a started session (`Live.subscribe` docstring: "Cannot be specified after the session is started"; iteration auto-starts). → v2 requires a *fresh* `db.Live()` client with `start=0` on the first subscribe, and accepts a "replay completed" `SystemMsg` as the connectivity pass.
- **`get_dataset_range` overclaim**: a returned range does not prove the paid plan is active (usage-based access also returns ranges) and says nothing about live licensing. → gate demoted to "historical access confirmed"; plan entitlement now rests on the portal Plans page, a `get_cost` = 0.0 probe, and the live subscribe itself.
- **Phase 7 contradicted the ≤15-line rule** (the verify script is 60–100 lines). → explicit two-exception carve-out (intake block, final file).
- **Live gate was undecidable when markets are closed** and could false-pass on `SymbolMappingMsg` floods (OPRA parent subscribes emit thousands of mappings even when closed). → record-type classification (only data records count), ~90 s wall-clock bound, and an explicit 4-row decision table with a provisional ✅\* state that must be converted during market hours.
- **Env-var visibility in the real launch context** (setx needs a new terminal; IDEs/Jupyter don't inherit shell profiles; conda) — the most likely original failure — was hand-waved. → per-OS/per-launcher recipes; gate 2 must run *the way the user actually runs Python*.
- **Interpreter/pip mismatch and PEP 668** unhandled. → `python`/`python3`/`py` triage, `<python> -m pip`, venv as the standard path on externally-managed environments, gate output pins `sys.executable`.
- **No SSL/proxy branch for the HTTPS side** (corporate TLS interception breaks pip and hist.databento.com before the live TCP issue ever appears). → dedicated troubleshooting branch (`pip-system-certs`, `REQUESTS_CA_BUNDLE`/`SSL_CERT_FILE`, IT allowlist).
- **No defense against "just give me the whole script"** — the first thing a previously-burned user says. → scripted one-sentence response + hard rule of one untested code block per message; multi-output pastes anchor on the earliest failed gate.
- **Licensing and billing never became gates.** → new gates 5 (subscriptions Active, questionnaires completed/not-pending — with a stop rule for pending) and 6 (PAYG + monthly limit + notifications), closed in Phase 4 before any billable call; personal-vs-professional use asked at intake because OPRA Standard is a personal plan.
- **The persisted verify script was a slow-burn billing leak** (hardcoded dates drift out of the rolling 12-month window; out-of-plan requests bill silently; re-runs re-bill streams). → dynamic date computation clamped inside `get_dataset_range`, an embedded `get_cost` abort (> $0.10), and runbook lines stating re-run costs and the ✅\*→✅ market-hours re-run.
- **Resumability too weak for a multi-day session.** → status header on every message + explicit resume rule.
- **"Look everything up in the repos first" collided with the prompt's own inlined facts.** → scoped: facts pinned to 0.78.0/0.62.0; repos consulted on gaps, contradictions, or version mismatch (CHANGELOG first).
- **Market-hours asymmetry** (OPRA 9:30–16:00 ET vs equities extended hours; Saturday test data; Sunday maintenance) could produce a false "OPRA broken" diagnosis. → market-clock step opens Phase 6 with expected per-feed behavior.

### Minor (all applied)

`limit=10` vs `limit=1000` contradiction explained per dataset · equities Standard live includes core (L0) schemas, not just top-of-book · PAYG-off behavior marked undocumented (hedged) · auto-revoke clarified as public-exposure-only · portal click-paths added (API keys, Billing) · Phase 5 date derived from `get_dataset_range` end (never today/weekend/holiday) · socket probe tests **both** gateways · 5-connections/s-per-IP retry note · gate 10 made observable (paste run + runbook text) · intake trimmed (env-var question replaced by the gate-2 command; version asked as `python`/`python3`/`py`) · partial-intake protocol · key-rotation follow-ons (live disconnects; redo gate 2) · pending-license stop rule.

## Final verification

A post-redraft verification pass re-checked v2's technical claims against the repo sources
and simulated executing the prompt for internal contradictions. Outcome and any residual
notes are recorded in the pull request description.
