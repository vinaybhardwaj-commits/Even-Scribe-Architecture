# Designer brief — speech turns on scratch (rev 3)
22 Aug 2026. Scribe Designer. For the orchestrator.

Rev 3 is written against production at `main` `9010ba8`, after your capability note. `stt_turn` is a cue type, not a table. Mini Whisper as wired returns one blob. Slice A changes that client, then posts turns to scratch.

Rulings are in `DESIGNER-REPLY-SPEECH-TURNS-CAPABILITY-22-AUG-2026.md`. Closed forks in `DESIGNER-REPLY-SPEECH-TURNS-REV3-ISSUES-22-AUG-2026.md`. This file is the build.

## In English

The goal of this phase has not changed. The brain must eventually say who is being treated, who the doctor is, and when an encounter starts, pauses, and stops. Later it must transcribe those encounters into a note, a timeline, a prescription, and a disposition.

We have a recording of two rooms from 19 August, and we have Pulse’s official log of who was billed that day. Those two stories barely match. So far the system has only read the log. It has never listened to the recording.

**We already used the notes.** Pulse does not store “Cardiology OPD.” It stores the doctor. Metabase reads the same tables. Checked 22 Aug: **Dr Dibyendu Majumdar has exactly one submitted note on 19 Aug. Zero drafts.** That consult is already on the Cardiology copy. Do not look for more. The five hours of tape with no note are not a missed join. Two extra kiosk taps have no note behind them.

This slice listens to two short clips, not the whole day:

1. The one Cardiology consult Pulse already knows — can we hear that visit on the tape.
2. A stretch of OPD 7 where the mic was on and Pulse has nobody — empty room, or a patient Pulse never logged.

The words land on a **copy** of the day. They do not invent a patient. They do not write a note.

## Slice A only

Widen `lib/whisper.ts` to request `verbose_json`. Widen `WhisperResult` so segments (`start`, `end`, `text`) survive the parse. Emit `cue` rows with `type='stt_turn'` onto the scratch `room_day`. Text and timings. No speakers.

Do not touch the encounter pipeline. Do not call `runDiarize`. Do not pull `lib/stt/eta-router.ts`. Do not create a `stt_turn` table. `0042` already allows the type.

First commit of A: blocklist on `scribe_post_cue` so `stt_turn`, `stt_silence`, `speaker_match`, `pqm_called`, `pstart`, `dx_event`, `pulse_note` cannot land on a live day. Producer types use the scratch path only (`room_day_id` + lock + `not_a_scratch_day` 409).

## What this is not

- Not slice B (clusters, `speaker_match`). After I see the two-window report.
- Not another Metabase join. Dibyendu’s 19 Aug count is closed: one.
- Not the live 250 ms ear. Not a second recorder. Not `NEXT_PUBLIC_ETA_LIVE_SINK`.
- Not a note, prescription, disposition, or clinician page.
- Not a live `room_day` write. Live OPD 7 stays one cue. Live Cardiology stays three kiosk marks.
- Not arms B or C.

## Already on scratch (do not redo)

Cardiology `rd_scratch_bh6jtq4t_20260819`: 3 visits. One bound at 06:54Z to the Metabase note. Two unknown marks (04:41Z, 09:34Z).

OPD 7 `rd_scratch_qyzghzaf_20260819`: 13 visits. Hole `04:36:59Z → 10:39:35Z` is a cue-timeline fact.

## Input

- OPD 7 `bs_xvntaugh` → `rd_scratch_qyzghzaf_20260819`
- Cardiology ended sessions → `rd_scratch_bh6jtq4t_20260819`
- `bs_8temrdqh`: skip by name. No backup, dead primary. Do not invent tape.

Windows ≤ 30 minutes. Refuse while any room is recording. Primary unless the session’s own mic events say primary was lost.

**Asked window, not the chunk.** If single-chunk mode still transcribes the whole five-minute piece, do not emit that as the window. Stitch or trim first.

## Emit

```
type        stt_turn
source      replay
at          segment start
source_ref  {session_id}|{start_ms}|{end_ms}|{speaker}
payload     { end, text, engine: whisper, source_used, window, language }
```

`source_ref` is four pipe-separated fields. `start_ms` / `end_ms` are integer epoch milliseconds, floored from Whisper’s clip-relative seconds onto the asked window start. `speaker` is `-` in slice A. Slice B writes the integer `speaker_idx` in that same slot. Silence uses the window start/end and `speaker` `-`.

One cue per Whisper segment. Blob-only response is a failed A — do not post one blob as a turn.

**Migration `0050`:** narrow `cue_replay_natural_key` to `WHERE source = 'replay' AND type NOT IN ('stt_turn', 'stt_silence', 'speaker_match')`. Add `UNIQUE (source_ref, type) WHERE source = 'replay' AND type IN ('stt_turn', 'stt_silence', 'speaker_match')`. Marks stay on 0046. Turn insert names that target. Report written / already existed / dropped.

Empty audio is `type = 'stt_silence'`, not a missing `stt_turn`. Same source, same scratch path.

Text is clinical speech. `scribe_list_cues` stays summary-only unless asked. Do not paste transcripts into Slack, chat, or this repo.

Fail closed if the provider is not Whisper. Do not write turns from a silent Ollama fallback. Report the language whisper.cpp picked (one language per file is a named prior).

## First demo, not 13 hours

1. Cardiology around 06:54Z (Dibyendu’s one note). Hear the consult that already has a visit.
2. OPD 7, one window inside `04:36:59Z → 10:39:35Z`. Words or an `stt_silence` cue. No new visit.

Then I read the report. Then the rest of the finished sessions on the same scratch days.

## Scoreboard

Same tool, `scribe_fuse_report`:

- `stt_turn` count
- `stt_silence` count
- tape minutes asked / minutes with words / minutes silent
- windows that used backup
- language whisper.cpp reported

`stt_turn` does not mint or close a visit. Arm A (rules) stays the visit writer.

## Acceptance

1. Dry-run lists turns, posts nothing. Live OPD 7 cue count stays 1. Live Cardiology stays three kiosk marks.
2. Scratch write: Cardiology 06:54Z window produces `stt_turn` rows with `payload.end` on `rd_scratch_bh6jtq4t_20260819`.
3. Scratch write: one hole window on OPD 7 produces words or `stt_silence`. It does not invent a visit.
4. Second write is 0 written, N already existed, 0 dropped.
5. `bs_8temrdqh` skipped by name.
6. `scribe_post_cue` of `type=stt_turn` or `stt_silence` on a live room is refused.
7. No Pulse writes. No Slack. No Gerrit. No second Dibyendu note.

Send the two-window report before walking the rest of the day.

## Slice B (not now)

`runDiarize` without `encounterId`. Write `speaker_cluster` from ECAPA. Then `speaker_match`. Weak. 0.78 banned. I write that brief after A.
