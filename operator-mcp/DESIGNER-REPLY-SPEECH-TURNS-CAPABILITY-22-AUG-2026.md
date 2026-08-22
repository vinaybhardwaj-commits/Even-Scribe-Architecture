# Designer reply — speech turns, against the real path
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-FINDINGS-SPEECH-TURNS-22-AUG-2026.md`. You asked for five rulings before a brief. Here they are. The brief is rewritten as rev 3 in `SPEECH-TURNS-SCRATCH-BRIEF-22-AUG-2026.md`.

Thank you for reading production first. The documents described a producer that the client cannot emit.

## Answers

**1. Turn shape — one cue.** `at` is the start. `payload.end` is the end. Do not mint two cues per turn. `cue` stays one clock.

**2. Cue key — do not unique on `(session_id, type, at)` alone for turns.** Two segments can share a start (and slice B will have two speakers in the same millisecond). Use `source_ref` the way warehouse does: `session_id + at + end` (and later `+ speaker_idx` in slice B). Partial unique on `(source_ref, type)` where `source = 'replay'`. The 0046 replay key stays for marks. Do not weaken it. Do not invent a turn table.

**3. Source value — `replay`.** The column is the insert path, not the modality. Replay of a finished tape is `replay`. `payload.engine = whisper` is the ear. Do not add `stt` to `CUE_SOURCES`. Live ear, if it ever posts, is a later source. Not this slice.

**4. `scribe_post_cue` — blocklist, not a full enum.** `cue.type` stays an open set. The tool must refuse producer types on a live day: `stt_turn`, `speaker_match`, `pqm_called`, `pstart`, `dx_event`, `pulse_note`. Operator may still post `operator_pin` and other operator evidence. Those producer types only land through the scratch path (`room_day_id` + lock + `not_a_scratch_day` 409), the same path `scribe_replay_write` uses. Do this as the first commit of slice A. It is the live-write hazard. Close it.

**5. Brief is rev 3, below.** Against `verbose_json` + scratch insert. Not against the blob `scribe_transcribe_range` returns today.

## Staging (V's ruling, accepted)

**Slice A now.** Widen `lib/whisper.ts` to `verbose_json`, widen `WhisperResult` with segments, emit `stt_turn` onto scratch only. Text and timings. No speakers. Do not touch the encounter pipeline. Do not call `runDiarize`. Do not pull the ETA router.

**Slice B after I have seen the two-window A report.** Bench caller for `runDiarize` that does not need `encounterId`. Write `speaker_cluster`. Then `speaker_match`. Weak by design. 0.78 banned. I will write that brief then.

`speaker_cluster` has no writer. That is why B is second, not parallel.

## Two limits you flagged, ruled

- **Whole-chunk vs asked window.** Posted turns must cover the **asked window**, not the five-minute slab. If single-chunk mode still returns the whole chunk, do not emit it as the window. Use the stitch path, or trim before insert. v1.1 trim is acceptable if it ships in A.
- **One language for the file.** Named prior. Code-mixed OPD can garble. Do not switch to the ETA router in A. Report the language whisper.cpp picked.

Metabase / Dibyendu still closed: one submitted note on 19 Aug. Do not re-join.
