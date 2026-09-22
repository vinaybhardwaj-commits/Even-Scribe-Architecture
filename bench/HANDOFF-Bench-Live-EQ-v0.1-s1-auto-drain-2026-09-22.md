# Handoff: Bench Live Equalizer v0.1 on `vinay/s1-auto-drain`

**Date:** 2026-09-22 IST  
**Author:** EvenScribe Designer (for transfer to another builder)  
**Status:** Shipped to www.evenscribe.app on branch `vinay/s1-auto-drain`

---

## 1. One-line summary

Added a live **VU / multi-bar microphone level meter** on every existing Bench room card, plus a durable **per-room IST-day level log**, without promoting `main` or bringing the fleet-board UI rewrite.

---

## 2. Why this landed on `s1-auto-drain`, not `main`

| Fact | Detail |
|---|---|
| Live production domain | `www.evenscribe.app` was intentionally pinned to **`vinay/s1-auto-drain`** (rollback after an accidental `main` deploy earlier on 22 Sep) |
| Live UI | Still the **classic** `BenchRoomsLive` cards (Main-mic piece-health vitals + start/pause/resume/stop on card) — **not** `BenchFleetGrid` / drawers from main |
| EQ first built on | `main` via ETA PRs **#11** (feature) + **#12** (migration renumber) |
| Why not promote main | Would have moved www onto fleet-board IA Vinay did not want live yet |
| What we did instead | Ported **EQ only** onto `s1-auto-drain` as PR **#13**, then pointed www aliases at that deploy |

**Do not assume `main` === production.** Production for Bench is currently the s1-auto-drain line unless aliases move again.

---

## 3. Product behavior (what operators see)

### 3.1 Card meter (`BenchLevelMeter`)

- Label: **Main microphone**
- State chip: `live` | `idle` | `digital silence`
- Visual: **18 vertical bars** (VU look), color bands blue → green → amber by height
- **Live when** listener is listening **and** room state is `recording` or `ready`
- **Idle (grey)** when finished / not listening / no levels — finished rooms still show the meter chrome, flat
- **Digital silence (red flat pattern)** when `zero_ratio >= 0.98` (or an explicit silence flag)
- Peak is visually scaled with `sqrt(peak / 0.3)` and **decays** between polls so ~1.5–3s heartbeats still feel continuous
- Spare mic still uses the old `LevelBar` only when `spare_exists && spare` levels exist (unchanged spare UX)

### 3.2 Level log (backend)

On each measured main-mic heartbeat in the command poll path:

- Upsert latest `mic_peak` / `mic_avg` / **`mic_zero_ratio`** on `bench_listener`
- **Append** a row to `bench_level_sample` (best-effort; failure never blocks the command bus)

Admin read API:

```
GET /api/admin/bench/levels?room_id=<id>&ist_date=YYYY-MM-DD
```

Returns 15-second display buckets: `peak`, `avg`, `zero_ratio`, `session_open`, `tape_advancing`, `samples`, `t_ms`. Admin cookie fence (`benchAdminGuard`).

### 3.3 Explicit non-goals of this port

- No `BenchFleetGrid` / fleet-first IA
- No room drawers / timeline sparkline UI on this branch (API exists; card meter is the live UX)
- No Room Recorder release required for v0.1 (uses existing poll fields; native can send `peak`/`zero_ratio` aliases)
- Does **not** mint visits from levels
- Does **not** stream raw audio to the browser

---

## 4. Git / deploy coordinates

| Item | Value |
|---|---|
| Feature PR (into s1-auto-drain) | https://github.com/vinaybhardwaj-commits/Even-Transcription-Assistant/pull/13 |
| Merge sha on `vinay/s1-auto-drain` | `248c2ae08908307f4cb46acc03e294b49c64ea3f` |
| Prior live tip (before EQ) | `de3e5ab` (Jev translate work) |
| Vercel deploy | `dpl_3JmsPyvwNYz3DUiJ7w3kCKy3xPH3` |
| Deploy URL | `even-transcription-assistant-mlgfp7e3j.vercel.app` |
| Aliases moved onto that deploy | `www.evenscribe.app`, `evenscribe.app`, `even-transcription-assistant.vercel.app`, `eta.llmvinayminihome.uk` |
| Previous production deploy (replaced) | `dpl_8Ze3LE18AYCeSc1r3yz5FTLhfpnR` @ `de3e5ab` |

Also on **main** (not serving www unless aliases change): PR #11 `0aec315` + PR #12 `069dc5d` (fleet-grid wiring + migration renumber). Treat main’s EQ as a parallel implementation aimed at the fleet board.

Architecture ticket / PRD:

- https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/13  
- https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/blob/main/bench/EvenScribe-Bench-Live-Equalizer-PRD-2026-09-22.md  

---

## 5. Database (already applied on prod)

**Neon project:** work org `eta-app-db` (`floral-butterfly-54959636`)

**Migration:** `db/migrations/0112_bench_level_samples.sql`  
**schema_migrations:** version **112**, name `0112_bench_level_samples`  
(Applied 2026-09-22 ~19:53 IST. Do **not** use version 69 — that slot is already `0069_build3_corrective`.)

DDL (idempotent):

1. `bench_listener.mic_zero_ratio real` (nullable)
2. Table `bench_level_sample` (`room_id`, `ist_date`, `sampled_at`, `peak`, `avg`, `zero_ratio`, `session_open`, `tape_advancing`, `source`)
3. Index `bench_level_sample_room_day_time_idx` on `(room_id, ist_date, sampled_at)`

If another environment needs the migration: run the SQL file as-is (`IF NOT EXISTS` / `ON CONFLICT DO NOTHING`).

---

## 6. Code map (files touched on s1-auto-drain via #13)

### UI

| Path | Change |
|---|---|
| `components/admin/BenchLevelMeter.tsx` | **New** — VU multi-bar meter component |
| `lib/bench-meter.ts` | **New** — `BenchMeterLevels`, `isDigitalSilence` |
| `components/admin/BenchRoomsLive.tsx` | Wire `BenchLevelMeter` into each room card (always render; live only when listening+recording/ready); spare still uses old `LevelBar` |

### Ingest / poll path

| Path | Change |
|---|---|
| `lib/bench-dual.ts` | `LevelAccumulator` now tracks `zero_ratio` (near-zero sample fraction using existing `SILENCE_RMS`) |
| `lib/use-command-poll.ts` | Sends `mic_zero_ratio` query param when present |
| `app/api/bench/commands/route.ts` | Accepts browser `mic_peak`/`mic_avg`/`mic_zero_ratio` **or** native `peak`/`zero_ratio` aliases into the same mic reading |
| `lib/bench-commands.ts` | Persist `mic_zero_ratio` on listener upsert; **append** `bench_level_sample` after poll (try/catch, never fail poll) |
| `app/api/admin/bench/listeners/route.ts` | Listener payload `mic` includes optional `zero_ratio` |

### Reads / schema / tests

| Path | Change |
|---|---|
| `lib/bench-levels.ts` | `MicLevels` extended; `readRoomLevelDay`, `isIsoDate`, bucket constant 15s |
| `app/api/admin/bench/levels/route.ts` | **New** admin timeline API |
| `db/migrations/0112_bench_level_samples.sql` | **New** migration (matches prod 112) |
| `tests/unit/bench-live-equalizer.test.ts` | **New** silence + migration version tests |
| `tests/unit/bench-commands.test.ts` | Append + `cleanLevels` coverage |
| `tests/unit/mic-health-b2.test.ts` | Expect `zero_ratio` from accumulator |
| `tests/unit/build3-recovery.test.ts` | Tiny string match update for spare `LevelBar` |

Approx size: **+457 / −34** across **15 files**.

---

## 7. Data flow (end-to-end)

```
Room Recorder / browser kiosk
  → LevelAccumulator (RMS → peak, avg, zero_ratio)  [bench-dual]
  → GET /api/bench/commands?...&mic_peak=&mic_avg=&mic_zero_ratio=
       (native may use peak=&zero_ratio=)
  → pollCommands upserts bench_listener
  → best-effort INSERT bench_level_sample (IST date)
  → GET /api/admin/bench/listeners → card props.mic
  → BenchLevelMeter (animate + decay)
```

Cadence: measure ~1 Hz on client; report on existing command poll (~1.5–3 s). UI poll for listeners ~few seconds.

---

## 8. Smoke checklist for the next builder

1. Hard-refresh https://www.evenscribe.app/admin/bench (confirm multi-bar **Main microphone**, not only the old single green piece-health strip)
2. Finished rooms → meter `idle` / grey
3. Start a listening room (or use one still recording) → bars should move with speech; decay between polls
4. Digital silence → red `digital silence` treatment when zero_ratio high
5. Card start/pause/resume/stop still present and unchanged
6. Optional: `GET /api/admin/bench/levels?room_id=…&ist_date=2026-09-22` as admin → samples accumulate after polls
7. Confirm DB: `SELECT version,name FROM schema_migrations WHERE version=112;` and `SELECT count(*) FROM bench_level_sample WHERE ist_date = CURRENT_DATE;` (or IST today)

---

## 9. Known follow-ups (not done)

| Item | Notes |
|---|---|
| **v0.2** higher-rate sample / activity intervals / FFT look | Needs Room Recorder release + product decision; PRD §6 |
| Timeline sparkline in UI on s1 cards | API exists; no drawer/sparkline wired on this branch |
| Reconcile `main` vs `s1-auto-drain` | Main has fleet-board EQ wiring; live has classic-card EQ. Next promote of main needs a deliberate plan |
| Rolling-window retention for `bench_level_sample` | Not implemented (table grows with every poll while rooms listen) |
| Close Architecture #13 | Partial: v0.1 live on s1; ticket still open for v0.2 |

---

## 10. Context the next builder should not lose

1. **Production branch ≠ main.** Aliases were manually assigned with Vercel `assign_alias` after `request_promote` returned 422 for a non-production-target preview deploy.
2. **Migration number is 112**, not 69.
3. Level append is **best-effort** by design — a missing migration must not brick room control; however prod already has 112, so appends should succeed.
4. Replacing the old conditional `LevelBar` for main mic: meter now **always renders**; live gating is inside the component via `live` prop.
5. Spare path intentionally left on legacy `LevelBar`.

---

## 11. Suggested first commands for the next builder

```bash
# Confirm branch tip
git fetch origin && git log -1 --oneline origin/vinay/s1-auto-drain
# Expect: 248c2ae Port Bench Live Equalizer v0.1…

# Diff of the port
gh pr view 13 --repo vinaybhardwaj-commits/Even-Transcription-Assistant
gh pr diff 13 --repo vinaybhardwaj-commits/Even-Transcription-Assistant
```

If extending meters: start at `components/admin/BenchLevelMeter.tsx` + the `BenchLevelMeter` call site in `BenchRoomsLive.tsx` (~near the former Main mic `LevelBar`). If extending logging/retention: `lib/bench-commands.ts` append block + `bench_level_sample`.
