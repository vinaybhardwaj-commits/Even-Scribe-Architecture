# The OPD 7 hole, walked

22 August 2026. From the orchestrator, to the Scribe Designer.
Prod `8820b5c`, migrations through `0053`.

K5 shipped and silence records itself. The six-hour warehouse blackout of 19 August,
`04:36:59Z → 10:39:35Z`, is now walked end to end in contiguous 15-minute windows.

**Twenty-five windows. Every one `complete: true`. Nothing failed. 1,708 segments of speech recovered
from six hours the warehouse has no record of.**

No live `room_day` written. No visit invented. No transcript in this document.

---

## 1. K5 verified first

The 07:45–07:51Z window that wrote nothing this morning now returns `silent_window: true`, one
`stt_silence` covering the window, and one `stt_window` marker with `complete: true` and
`segment_count: 0`. The ask finished and the day says so.

The builder's solve is better than the spec: `empty_transcript` builds a turn-set from zero segments
and lets the existing rule produce the row, so a silent window and a window whose every segment was
blank emit the identical cue. Both Whisper call sites route through one function. The two paths cannot
drift.

## 2. The shape of the six hours

Segment counts from the `stt_window` markers, which are the ground truth rather than any report.

| IST | UTC | segments | character |
|---|---|---|---|
| 10:07–10:22 | 04:37 | 237 | busy |
| 10:22–10:37 | 04:52 | 160 | consultation, then a billing dispute |
| 10:37–10:52 | 05:07 | 5 | thin |
| 10:52–11:07 | 05:22 | 60 | staff, then a phone call |
| 11:07–11:22 | 05:37 | 1 | thin |
| 11:22–11:37 | 05:52 | 374 | consultation and a phone call |
| 11:37–11:52 | 06:07 | 103 | consultation |
| 11:52–12:07 | 06:22 | 94 | staff |
| 12:07–12:22 | 06:37 | 407 | staff, count inflated by a loop |
| 12:22–12:37 | 06:52 | 1 | thin |
| **12:37–12:52** | **07:07** | **0** | **true silence** |
| 12:52–13:07 | 07:22 | 1 | thin |
| 13:07–13:22 | 07:37 | 1 | thin |
| 13:22–13:37 | 07:52 | 1 | thin |
| 13:37–13:52 | 08:07 | 1 | thin |
| 13:52–14:07 | 08:22 | 3 | thin |
| 14:07–14:22 | 08:37 | 1 | thin |
| 14:22–14:37 | 08:52 | 55 | staff, mostly loop |
| 14:37–14:52 | 09:07 | 81 | mostly loop, no clinical content |
| 14:52–15:07 | 09:22 | 22 | staff, about the recording |
| 15:07–15:22 | 09:37 | 67 | about 90% hallucinated filler |
| 15:22–15:37 | 09:52 | 2 | filler |
| 15:37–15:52 | 10:07 | 21 | background or distant speech |
| 15:52–16:07 | 10:22 | 4 | filler |
| 16:07–16:10 | 10:37 | 6 | consultation, history taking |

**Three phases.** A working morning clinic from 10:07 to about 12:22 IST, carrying real
consultations. A dead stretch of roughly an hour and three quarters over lunch, 12:22 to 14:07. Then
an afternoon that reads busy in the counts and is mostly not.

**The morning is the finding.** Consultations, a phone call, and one extended conversation with a
family about an ICU bill in which the clinician reasons aloud about which investigations were
necessary and which were deliberately not ordered. None of it has a warehouse event.

## 3. The defect the walk exposes

**Whisper rarely returns an empty transcript on quiet audio. It hallucinates instead.**

Of roughly eight windows that were effectively quiet, exactly **one** returned an empty transcript and
produced an `stt_silence`. The other seven returned one to three turns of filler — a single generic
token stretched across ten minutes, or one short phrase repeated. Two windows were reported as 100%
repeated identical filler. One was 49 of 81 turns as a single repeated sentence.

So the type you designed works, and it fires on about one in eight of the windows it exists for. The
rest of the quiet time is now recorded as thin speech, and **nothing downstream can tell a
one-turn hallucination from a genuinely brief exchange.**

This is not a bug in K5. K5 does exactly what it was asked. It is the transcriber refusing to admit
silence, and the greedy decoder has reduced it without removing it.

We are not proposing a fix. It needs your ruling, and the obvious candidates each have a cost: a
minimum-content heuristic invents a threshold, and a VAD pre-pass adds a second stack you have
forbidden.

## 4. A second finding: overlapping windows do not deduplicate

We expected the four earlier probe windows to dedupe against the contiguous walk that overlaps them.
They did not. `already_existed` was **0 on every window of the walk**.

The reason is sound and worth writing down. A different asked window produces a different joined clip,
and a different clip makes Whisper segment differently. Different boundaries mean a different
`source_ref`, so the same speech lands twice under two keys.

**Consequence:** up to 289 turns from the four probes are duplicated on the day under different
window attributions. Harmless for reading, wrong for counting.

**Recommendation:** make non-overlapping windows a rule for any future walk. The window is the write
unit, and two windows covering the same audio are two different asks, not one.

## 5. What held

Twenty-nine markers on the day, every one `complete: true`. Across the whole walk: **zero `failed`,
zero windows rolled back, and two `dropped`** — both in one window, both Whisper emitting the same span
twice inside a single write, which is what `dropped` now correctly means.

The window-as-unit writer, the marker upsert, the count split and the silence path all did their jobs
across twenty-five consecutive production writes.

## 6. What we ask

1. **Rule on hallucinated silence.** One in eight is the current coverage of `stt_silence`.
2. **Ratify non-overlapping windows** as the rule for walks, and say whether the 289 duplicated probe
   turns should be deleted or left as an honest record of how they were asked for.
3. **19 August OPD 7 can now be treated as heard**, subject to those two. Cardiology's finished
   sessions are not walked yet — say whether they come before or after Monday's recording.

Monday's questions are still open in
`ETA-MONDAY-READINESS-AND-OPEN-RULINGS-22-AUG-2026.md`. Consent copy and the second microphone remain
the two that cannot be repaired once the day starts.
