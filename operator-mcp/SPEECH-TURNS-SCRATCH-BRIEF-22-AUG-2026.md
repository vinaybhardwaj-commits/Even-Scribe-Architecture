# Designer brief — speech turns on scratch
22 Aug 2026. Scribe Designer. For the orchestrator.

This is the next build. It is not another warehouse slice.

## Why

The phase goal has not changed. The brain must eventually say who is being treated, who the doctor is, and when an encounter starts, pauses, and stops — then transcribe those encounters into a note, a timeline, a prescription, and a disposition.

§11 gave us a visit graph from Pulse’s clocks and a scoreboard that shows those clocks barely overlap the tape. The brain has still never heard the room. Until `stt_turn` exists, “who” is only a Pulse id, and “when” is only a Pulse timestamp. That is matching Pulse to itself.

This slice puts words on the scratch graph. Notes, prescriptions, and disposition are **not** this slice.

## What to build

A producer that walks a **finished** Bench session, hears the tape with the plumbing that already exists (`scribe_extract_audio` / `scribe_transcribe_range`, Mini Whisper, U2 stitch, U4 backup-when-primary-lost), and posts `stt_turn` cues onto the **scratch** `room_day` for that session’s real room.

§9 already named this emit. Nothing emits it. `0042` already has the type. Do not invent a second STT stack.

## What this is not

- Not the live 250 ms ear. Not a second recorder. Not `NEXT_PUBLIC_ETA_LIVE_SINK`.
- Not `speaker_match` as a binder. Voice on this corpus scored 0.55–0.62. 0.78 is banned. Local ear labels may ride on the payload if diarization is free; they do not become `individual_uid` or “this is the doctor.”
- Not a note writer. Not a prescription. Not a disposition. Not a clinician page.
- Not a write to any live `room_day`. Live OPD 7 stays one cue. Live Cardiology stays three kiosk marks and zero visits.
- Not arms B or C. Do not re-run them.
- Not promotion.

## Input

The 19 Aug corpus, scratch days already fused:

- OPD 7 `bs_xvntaugh` → `rd_scratch_qyzghzaf_20260819`
- Cardiology ended sessions → `rd_scratch_bh6jtq4t_20260819`

`bs_8temrdqh` has no backup and a dead primary. Do not invent tape. Skip with a named reason.

`transcribe_range` already refuses windows over 30 minutes and refuses while any room is recording. Keep both. Walk a long session in windows ≤ 30 minutes. Primary unless the session’s own mic events say primary was lost.

## Emit

Cue type `stt_turn`. Column `source = 'replay'`. Payload provenance stays honest (`engine`, `source_used`).

```
at          start of the turn (Whisper segment time if you have it, else window start)
session_id  the Bench session
text        the words
engine      whisper
source_used primary | backup
window      start/end asked
```

If Whisper returns segments, one cue per segment. If it returns a blob, one cue per window. Do not drop empty audio — post nothing and count `no_audio_in_range` on the report.

Idempotency is the replay key you already have: `(session_id, type, at)` where `source = 'replay'`. Re-running a session does not double the words.

Text is clinical speech. `scribe_list_cues` stays summary-only unless `include_payload` is asked. Do not paste transcripts into Slack, chat, or this repo.

## First demo, not the whole 13 hours

Do not start by transcribing both rooms end to end. Two windows, then the rest:

1. **A consult Pulse already knows.** Cardiology, around the 06:54Z `pstart` (the one billed consult). We should hear a consult that already has a visit.
2. **The hole.** OPD 7, a window inside `04:36:59Z → 10:39:35Z` where tape was running and the warehouse has nothing. We should hear whether the room was empty or an unmarked consult was happening.

Those two answers are the product question. After they land and I have read them, walk the rest of the finished sessions onto the same scratch days.

## Scoreboard

Extend `scribe_fuse_report` (same tool, not a new one):

- count of `stt_turn` on that scratch day
- tape minutes asked / minutes with words / minutes `no_audio`
- which windows used backup

Do not let `stt_turn` mint or close a visit in this slice. Arm A stays the visit writer. Words are evidence. The next brief will say when the fuse may use them.

## Acceptance

Against scratch only:

1. Dry-run lists the turns and posts nothing. Live OPD 7 cue count stays 1. Live Cardiology stays three kiosk marks.
2. Write on scratch: Cardiology window around 06:54Z produces `stt_turn` rows on `rd_scratch_bh6jtq4t_20260819`.
3. Write on scratch: at least one window inside the OPD 7 hole produces either words or an explicit `no_audio` count. It does not invent a visit.
4. Re-run is idempotent. Second write is 0 new turns.
5. `bs_8temrdqh` is skipped by name, not faked.
6. Fail closed if Whisper is not actually Whisper. Same provider lock as the bake-off. Do not write turns from a silent Ollama fallback.
7. No Pulse writes. No Slack. No Gerrit. No live `room_day`.

Send the two-window report here before walking the rest of the day.

## After this (not now)

`speaker_match` as a weak prior. Then a fuse pass that may use words to bind an `unknown` mark or to say “the hole was empty.” Then, much later, the note / timeline / prescription / disposition pipeline that **reads** this graph. Pulse still commits.

A second clinic day is still the highest-value *input*. It is not a blocker for the two-window demo on tape we already have.
