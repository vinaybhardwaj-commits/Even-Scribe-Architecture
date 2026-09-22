# Bench Live Equalizer / Level Meter — PRD
**Date:** 2026-09-22 IST  
**Owner:** EvenScribe Designer  
**Product:** Even Scribe Bench (`/admin/bench`)  
**Repos:** Even-Scribe-Architecture (ticket), Even-Transcription-Assistant (code)

## 1. Problem

Ranch / clinic operators watching Bench cannot tell in real time whether each kiosk mic is hearing **someone talking**, **background noise**, or **digital silence**. Bugs like “says recording, silent” and USB dead-path only show up as coarse chips after the fact. Encounter clocks also lack a cheap “was the room active?” timeline without STT.

## 2. What already exists

Room Recorder heartbeats already include:

- `peak` — instantaneous peak level (0–1 scale)
- `zero_ratio` — fraction of near-zero samples (digital silence detector)

These land in fleet / room-facts today (confirmed live 2026-09-22). Bench uses them for silence chips in places, but:

- no elegant continuous visualizer per card
- no durable per-room level history (only latest snapshot)

## 3. Goals

1. Live equalizer-style visualizer on every room card while the mic/install is live.
2. Continuous level log, one stream per room (per IST day), separate from tape/STT.
3. Enable later “activity intervals” as encounter *candidates* (not automatic visit minting).

## 4. Non-goals (v0.1)

- Streaming raw audio or waveforms to the browser
- Minting visits / Pulse notes from level alone
- Replacing STT, diarize, or centroid ID
- Full FFT spectrum analyzer as a hard requirement (optional polish)

## 5. Product design

### 5.1 Live meter (card)

- Compact multi-bar meter under room name / status row.
- Driven primarily by `peak`; `zero_ratio` high → flat / “digital silence” treatment even if peak flickers.
- Color bands (tunable): quiet floor / ambient / speech-energy.
- Motion: smooth decay so heartbeat cadence still feels continuous.
- States when not live: grey idle (day finished, no poll, DEVICE_MISSING, mic unauthorized).

### 5.2 Room drawer

- Larger meter + **today’s level timeline** (sparkline of logged samples).
- Hover/scrub shows IST time + peak/zr.
- Optional: list of auto-detected activity islands (v0.2).

### 5.3 Level log (per room)

On each install heartbeat ingest, append:

```
{ t_ms, peak, zero_ratio, input_device_name?, session_open?, tape_advancing? }
```

Retention: keep current IST room-day hot; older days cold/archive policy TBD (start: 14–30 days).

Separate store from tape pieces and STT windows. No PHI audio bytes.

### 5.4 Encounter detection (v0.2+)

From the log, mark intervals where peak stays above speech threshold for ≥ N seconds, with hysteresis. Surface as **candidates** on the day timeline. Do not mint visits.

## 6. Phasing

| Phase | Ship | Needs RR release? |
|---|---|---|
| **v0.1** | Card meters + server append of existing heartbeat peak/zr + drawer sparkline | No |
| **v0.2** | Higher-rate local sample (≈10 Hz), 1 Hz durable log, activity candidates, optional band EQ | Yes |

## 7. Acceptance (v0.1)

- Live rooms show meters that move with real peak while installs poll.
- Digital silence (high zero_ratio / near-zero peak while “recording”) is visually distinct.
- Per-room day log accumulates samples; drawer shows timeline.
- Admin-only; no clinician/kiosk UI change required for v0.1.

## 8. Open questions for Vinay

1. Prefer **simple VU / bar meter** (recommended) or literal **FFT band equalizer** look?
2. Log retention: 14 days vs 30 days hot?
3. Start build on ETA now (v0.1) without waiting on a Room Recorder bump?

## 9. Success

An operator can open Bench mid-clinic and, in under a second per room, see whether that mic is quiet, noisy, or carrying speech-energy — and later scrub that room’s day for when activity happened.
