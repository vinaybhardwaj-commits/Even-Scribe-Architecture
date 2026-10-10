# Builder brief: capture / fleet gaps (5–6 Oct 2026)

**Date:** 6 Oct 2026  
**Audience:** ETA builder  
**Status:** Spec only. This repository does not hold the implementation.  
**Tickets:** [Architecture #14](https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/14), [#15](https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/15), [#16](https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/16), [#17](https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/17)  
**Suggested build order:** #14 → #15 → #16 → #17

---

## Where this gets built

Product work lands in [Even-Transcription-Assistant](https://github.com/vinaybhardwaj-commits/Even-Transcription-Assistant) (Room Watchdog, room-facts, Bench, kiosk copy, and the Room Recorder surfaces that ship from that product). Architecture keeps the tickets. Do not implement these gaps in Even-Scribe-Architecture.

Host USB / coreaudiod remount, waking a sleeping Mini, and per-room hardware RCA stay with **Kiosk Bot** and **Bug Bot**.

## Fleet context (6 Oct 2026)

| Room | Fleet status |
|---|---|
| Home Office Mini | **Retired.** Do not treat it as a live fleet room. |
| Dietary | **Still in the fleet.** Stale-Recording and auto-start evidence that names Dietary is still in scope. |

---

## Arch #14 — SILENT digital-zero vs quiet OPD

**Ticket:** https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/14

### Problem

`SILENT_WHILE_RECORDING` fails to tell **exact digital capture silence** from a **quiet or empty OPD**.

An empty clinic still has ambient (AC, corridor, HVAC). Samples from a live mic are not bit-exact zeros. **Tape advancing + near-zero peak + `zero_ratio` ≈ 1** means the capture path is dead or muted. Product and ops have been reading that pattern as “empty room,” and the watchdog has left it unflagged.

On 0.1.22+ the poll’s `silence_ms` (time since a sample above −55 dBFS) has been allowed to decide the flag. That clock is the wrong signal: a quiet live room can sit under −55 dBFS, and the Room Recorder PCM tail meter under-states the clock (a read covers at most the last 30 seconds; a longer gap or a new file restarts it), so a short `silence_ms` can veto a long `zero_ratio` run.

The Bench live EQ meter already treats `zero_ratio >= 0.98` as digital silence. The alert path must use that same ratio.

### Intended fix

Raise `SILENT_WHILE_RECORDING` (Room Watchdog, and any room-facts / Bench silence chip that mirrors it) when **all** of these hold:

1. The session is **recording**.
2. The **tape is advancing**.
3. `zero_ratio >= 0.98` is sustained for **about 2 minutes** (align the ratio with the Bench meter; 0.98, not 1.0, so one odd non-zero glitch does not hide a dead input).

Rules around that window:

- A reported checkpoint **peak `>= 0.01` breaks the run**. That is a live microphone. The 5 Oct speak-test band on a working C270 was 0.012–0.026; 0.01 sits just under it. A poll that omits peak makes no claim, and the ratio decides alone.
- Speak energy (peak at or above 0.01, or `zero_ratio` under 0.98) resets the run and clears the flag.
- **`silence_ms` neither raises the flag nor vetoes a digital-zero run.** Keep storing it. Do not consult it for this flag.
- Operator copy says **digital silence** (the meter’s existing words). Watchdog / degraded text names capture death: exact zeros on the capture, distinct from empty-clinic quiet. Suggested degraded phrase: “digital silence on the capture (exact zeros, not a quiet room).”

The meter may keep painting “digital silence” from the ratio alone on the current slice. The **alert** adds the peak conjunct the meter does not: a reported peak `>= 0.01` breaks the run, so a speak-alive peak is not paged as capture death. Record that delta if the meter and the alert stay different.

### Acceptance

- [ ] Recording + tape advancing + `zero_ratio >= 0.98` sustained ~2 minutes raises the silence / capture-dead alert inside that window. A multi-hour miss is a failure.
- [ ] A reported peak `>= 0.01` breaks the run and does not fire. Quiet-but-alive ambient (nonzero floor, speak-test peaks 0.012–0.026) does not fire as digital silence.
- [ ] `silence_ms` does not raise the flag and does not veto a qualifying digital-zero run.
- [ ] Alert copy and room-facts vocabulary say **digital silence** and distinguish capture death from empty-clinic quiet.
- [ ] The ratio stays aligned with Bench `BenchLevelMeter` (`zero_ratio >= 0.98`) unless a deliberate delta is written down. The peak floor (0.01) is the allowed delta versus the meter.
- [ ] Regression coverage: (a) exact zeros + advancing tape → fire, including when `silence_ms` is short; (b) low ambient above the floor, including a long `silence_ms`, → no fire; (c) speak energy resets the run and clears the flag.

### Out of scope

- Host USB / coreaudiod remount (Kiosk Bot / Bug Bot).
- Replacing the VU / live EQ meters themselves.
- Day-boundary and the 15s / 0-chunk start race (Arch #15 and #17).

### Evidence

| When (IST) | What |
|---|---|
| 5 Oct 2026, OPD5 | Session `bs_hqn9var3`, mic C270 `8132B850`. Stretches of up to **~5 minutes** of digital zeros (`zero_ratio` ~1, peaks ~0) while the tape kept advancing. **SILENT never fired.** Also flagged earlier the same day at 09:57 and 11:54. Speak-test ~16:28–16:31 proved the mic alive (peaks **0.012–0.026**). |
| 6 Oct 2026, ~10:55, OPD5 | Session `bs_r4cn5vmq`. **2 hours+** of digital-zero while recording, and **SILENT still did not fire.** |

Bug Bot product push 5 Oct 2026 (~16:39 IST). Related Bench surfacing: Architecture #8 (digital silence chip), #12 (promote silence to attention).

### Optional reference draft

https://github.com/vinaybhardwaj-commits/Even-Transcription-Assistant/pull/14

That draft is one sketch of this contract (carried digital-zero count, shared 0.98 ratio, peak 0.01 breaks the run, `silence_ms` stored and ignored, chip copy “digital silence”). It is **optional**. The builder may adopt it, rework it, or rewrite it. This brief is the spec; the draft is not.

---

## Arch #15 — Hard start ≠ “Finished for today”

**Ticket:** https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/15

### Problem

A hard tapewriter death is painted as a normal end of day. After start failures the kiosk showed **“Finished for today · Press start to record again”** — healthy EOD copy — so operators lose the signal to remount or retry.

### Intended fix

Failed-start UX for tapewriter **exit 1 / ~15s / 0 chunks** and any equivalent hard capture fail:

- Surface **failed start** (or the same meaning): capture did not start, with copy that invites retry / remount when that is the right next step.
- A **true EOD** finish (operator stop or scheduled end) keeps the existing **Finished for today** vocabulary.
- Bench / fleet must not show a healthy-EOD state when the session died at start. Room-facts that mirror the kiosk copy follow the same split.

### Acceptance

- [ ] Tapewriter exit 1 / ~15s / 0-chunk / hard capture fail surfaces as failed start, not “Finished for today.”
- [ ] Copy invites retry / remount when that is appropriate.
- [ ] True EOD finish still uses Finished-for-today vocabulary.
- [ ] Bench / fleet does not show healthy-EOD when the session died at start.

### Out of scope

- The underlying CoreAudio / device-not-ready race (Arch #17).
- Host remount ops (Kiosk Bot / Bug Bot).

### Evidence

5 Oct 2026 Bug Bot EOD open watch (b), and the same-day Designer note on OPD4 UI after hard exits.

**OPD4 ×12:** tapewriter **exit status 1**, about **15 seconds**, **0 chunks**, and the kiosk showed Finished-for-today EOD copy.

---

## Arch #16 — Offline kiosk stale cloud Recording

**Ticket:** https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/16

### Problem

An offline kiosk keeps a **stale cloud Recording** state. The brain says Recording and the operator board can look live after the host is gone. Kiosk liveness and cloud session state are desynced.

### Intended fix

Documented **TTL / heartbeat**: when the kiosk stops polling or the host goes offline (including sleep), cloud session state **leaves Recording** or is marked **stalled / offline** inside that window.

- Bench / fleet surfaces **host-offline / desync** for these rooms (ties to Architecture #8 / #12 chips). A healthy Recording chip is the wrong picture.
- Cardiology-class and Dietary-class stale Recording cannot sit for hours with no attention signal.
- Recovery when the kiosk returns is clean: resume or reconnect without a duplicate day session unless a new session is intentional.

### Acceptance

- [ ] Documented TTL / heartbeat: no kiosk poll → cloud leaves Recording, or the session is marked stalled / offline.
- [ ] Bench / fleet shows host-offline / desync for these rooms.
- [ ] A stale Recording of the Cardiology / Dietary kind cannot persist for hours without an attention signal.
- [ ] Return of the kiosk recovers cleanly (no duplicate day session unless intentional).

### Out of scope

- Waking sleeping Minis (ops / Kiosk Bot).
- Dietary or Cardiology hardware RCA.

### Evidence

5 Oct 2026 Bug Bot EOD open watch (c). Related Bench P0.4 / P0.7 desync chips.

| Room | Stale cloud Recording since (IST, 5 Oct) | Actual state |
|---|---|---|
| Cardiology | ~19:51 | Host offline; not capturing |
| Dietary | ~16:04 | Host offline; not capturing |

Dietary is still in the fleet. Home Office Mini is retired and is not part of this fix.

---

## Arch #17 — Fleet auto-start 15s / 0-chunk race

**Ticket:** https://github.com/vinaybhardwaj-commits/Even-Scribe-Architecture/issues/17

### Problem

Fleet **auto-start ~09:06 / 09:07 IST on 5 Oct 2026** produced **~15s / 0-chunk tapewriter deaths on five Minis at once**. OPD4 failed repeatedly; the same signature also hit Dietary / S2 and OPD3. The working hypothesis is a shared start-up race or capture-device-not-ready. That hypothesis is unverified until a repro morning’s logs confirm or revise it.

Deaths are silent: no alert, and the UX can read as Finished for today (that copy split is Arch #15). A simultaneous start must not cascade a shared device-not-ready failure with no retry.

Related, and **not** this ticket’s must-fix: auto-start can run ~18.5h, and a stalled session only dies via the 30-minute no-chunk reaper. A fuller day-boundary redesign may follow later.

### Intended fix

Before a day session is committed as Recording:

- Check that the **input device is ready**, or **defer / retry with backoff** until it is.
- Classify and **alert** a 15s / 0-chunk death. It is not silent, and it is not “Finished for today” (pair with #15).
- A simultaneous multi-room start does not cascade shared device-not-ready failures without retry.
- Validate or revise the device-not-ready hypothesis against **logs from a repro morning**. Do not ship the hypothesis as fact.

An early stall signal for 0-chunk / no-piece just after start is in scope only as far as it serves that alert. The 18.5h / day-boundary redesign stays a follow-on.

### Acceptance

- [ ] Auto-start checks input-device ready, or retries with backoff, before committing a day session.
- [ ] A 15s / 0-chunk death is classified and alerted (not silent; not “Finished for today” — see #15).
- [ ] Simultaneous multi-room start does not cascade shared device-not-ready failures without retry.
- [ ] The device-not-ready hypothesis is validated or revised with logs from a repro morning.

### Out of scope

- Per-room USB remount ops (Kiosk Bot / Bug Bot).
- Full day-boundary redesign (possible follow-on; the ~18.5h auto-start / 30-minute reaper note stays recorded, not built here).
- The Finished-for-today copy change itself is #15; this ticket must not keep emitting that copy for a start death, and #15 owns the vocabulary.

### Evidence

Bug Bot 5 Oct 2026 EOD (d). Designer notes the same day on the fleet auto-start race / S2 signature.

**09:06 auto-start** hit **five Minis at once** with **~15s / 0-chunk** deaths (OPD4 repeated; Dietary / S2 and OPD3 on the same pattern).

---

## Build order

1. **#14** — digital silence vs quiet OPD. The 6 Oct OPD5 miss (`bs_r4cn5vmq`, 2h+) is the live failure.
2. **#15** — failed-start copy vs Finished for today. Operators currently cannot see an OPD4-class death.
3. **#16** — stale cloud Recording while the kiosk is offline.
4. **#17** — auto-start device-ready / retry, and an alert on 15s / 0-chunk. Depends on #15 for the vocabulary of a start death.

Each ticket can ship on its own. Later tickets should not reopen the copy and flag rules of earlier ones.
