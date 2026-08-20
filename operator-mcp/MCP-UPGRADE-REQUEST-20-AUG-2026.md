# Operator MCP upgrade request
20 Aug 2026. Scribe Designer. For the builder orchestrator.

Cut coder briefs yourself from this. Product work stays on `feat/operator-mcp` in `Even-Transcription-Assistant`. Design lives in `Even-Scribe-Architecture`. Designer does not push to the product repo. Do not merge until Vinay says the 19 Aug tapes are safe.

Live door: production `385aa6b`, 33 tools, connected and used from this chat. Remount is accepted. This request is what the door still cannot do.

---

## Ground (measured today)

| Room | Session | What the tape is |
|---|---|---|
| OPD 7 / Ankit | `bs_xvntaugh` | 86 verified chunks, ~7 h, 416 MB, no backup stream |
| Cardiology / Dibyendu | `bs_vqummntv` | 41 chunks, “Cardio OPD Dr. Dibyandu” |
| Cardiology / Dibyendu | `bs_j9wgfa33` | 10 chunks, kiosk crash; `ended_at` is the **manual** close, not last audio (`last_chunk_at` is the tape clock) |
| Cardiology / Dibyendu | `bs_8temrdqh` | 20 chunks, label “mic stopped working”, **zero** backup chunks |
| OPD Test (yank) | `bs_wy5yjj7a` | backup chunks exist, `primary_lost_count` 4, `mic_status` `on_backup` |

The 33 already covers watch, start/stop/pause/mark, pin-as-cue, extract-by-clock (one chunk), transcribe-whole-piece. Do not add more start/stop verbs.

---

## Requested slices (this order)

### U1 — Replay dry-run

`scribe_replay_session` on a finished Bench session.

| | |
|---|---|
| Default | Dry-run. Emit the cue list. **Post nothing.** |
| First target | `bs_xvntaugh` |
| Emit | `consult_mark` from `bench_event`, `stt_turn` later if the slice includes STT; timestamps from chunk clocks, not `ended_at` |
| Idempotent | Re-run does not duplicate (natural key: session + type + at) |
| Out | No live `room_day` write. Scratch is U1.1 / fuse, not this slice |

Acceptance: dry-run on `bs_xvntaugh` returns a cue list; `scribe_list_cues` for that room-day is unchanged.

This is still the fuse unlock. Do it first.

### U2 — Hear a consult, not a chunk

Two limits in the live door:

1. `scribe_extract_audio` answers `multi_chunk_not_supported_v1` if the window crosses two 5-minute pieces.
2. `scribe_transcribe_range` transcribes the **whole** covering piece, not the window.

A real consult always crosses a chunk boundary. Ankit’s day is 86 pieces.

| Change | Rule |
|---|---|
| Extract | Stitch covering chunks into one short-lived clip (Mini ffmpeg is the v1.1 default already ratified). Return one presign + the requested IST bounds. |
| Transcribe | Run Whisper on that clip (or trim after). Return text for the **window**, not the 5-minute slab. |
| Still | Never inline bytes. `source` primary/backup remains. |

Acceptance: extract and transcribe `bs_xvntaugh` for a 12-minute window that straddles two chunks. One clip. Text covers that window. No `multi_chunk_not_supported_v1`.

### U3 — One day picture

Assemble by hand is possible and too slow.

| Tool | Returns |
|---|---|
| `scribe_day_report` | One room, one IST day: sessions (id, status, `started_at`, **`last_chunk_at` as tape end**, `ended_at` if different), chunk counts primary+backup, gaps, consult marks, mic events, `kiosk_remount_resumed` rows. Tape clock, not row clock. |
| `scribe_diff_room` | Listeners vs recording vs last cue vs last chunk. Flags: `kiosk_not_listening`, `tape_without_cues`, `stalled`, `ended_at_lies` (ended_at ≫ last_chunk_at). |

Acceptance: day report for OPD 7 on 2026-08-19 names `bs_xvntaugh` with tape end = `last_chunk_at`. Day report for Cardiology that day shows three sessions and flags `bs_j9wgfa33` as crash/manual end.

### U4 — Backup-aware extract

Yank tests already have a backup stream. Dibyendu’s “mic stopped working” tape (`bs_8temrdqh`) has **no** backup — dual-mic was not on that session. When backup **exists**, the door should hear it without the operator naming `source:backup`.

| Mode | Rule |
|---|---|
| Default | `source` omitted → primary. If that window is silence / missing and backup chunks exist, return backup and say so (`source_used: backup`, reason `primary_silent` or `primary_missing`). |
| Explicit | `source: primary` or `source: backup` still wins. |
| No backup | Today’s error (`no_audio_in_range`). Do not invent audio. |

Acceptance: extract a window on `bs_wy5yjj7a` where primary was lost; response uses backup and says so. Extract on `bs_8temrdqh` with no backup still fails clean.

---

## Do not

- Merge `feat/operator-mcp` to `main` until Vinay says the 19 Aug tapes are safe.
- Add start/stop/pause verbs. Control is enough.
- Flip `NEXT_PUBLIC_ETA_LIVE_SINK` in prod.
- Change the 1.5 s poll interval.
- Touch `/api/admin/r2-cors-fix`.
- Add kiosk MB-after-rejoin sizes.
- Write Pulse. Post Slack.
- Build a native always-on room agent.
- Stream 250 ms audio through MCP.
- Copy this design into the product repo to fix README links. Architecture repo is the home.
- Ask the designer for a file-by-file patch list.

Token-in-path is a password that appears in request logs. Not a tool. Prefer Bearer (already how this designer is connected). Rotate if logs are shared. Out of these four slices.

---

## What I will review

- U1: cue list for `bs_xvntaugh`, production cues untouched.
- U2: one 12-minute straddling window, one clip, window-shaped transcript.
- U3: Cardiology 19 Aug report uses `last_chunk_at` for `bs_j9wgfa33`.
- U4: yank session falls back; no-backup session does not pretend.

Send the preview report back to Vinay → this designer.
