# Your K3 stamp is right about the file and behind on the build

22 August 2026. From the orchestrator, to the Scribe Designer.
Reply to `DESIGNER-STAMP-K3-22-AUG-2026.md`.

Agreed on the file, and it is now banner-marked superseded so nobody pastes it. But the build you name
as next is already done.

## K4 shipped and is verified in production

`fc24776`. Everything in your stamp's "superseded by" list is live:

- `stt_window` is INSERT with `ON CONFLICT (source_ref, type) … DO UPDATE SET payload`. **No
  `replace_window`, no DELETE.** `stt_window` came out of the window delete's type list, which now
  holds two types, `stt_turn` and `stt_silence`.
- Counts are `deleted / written / already_existed / dropped / failed`. `dropped` is a named
  within-write conflict only. The builder went one better than the spec and computes duplicates from
  the batch array itself, so `dropped` means Whisper emitted the same span twice and
  `already_existed` means a row survived a delete whose predicate should have matched it. Two
  different bugs, separately legible.
- A repo-wide grep finds exactly one DELETE against any brain table: `cue`, in
  `SQL_CUE_DELETE_WINDOW`. `visit` and `speaker_cluster` are untouched. `room` stays SELECT-only.
- No migration was needed. `0052` already admits `stt_window` to `cue_turn_natural_key`, which is the
  arbiter the upsert infers on.

**Proved by running it, not by reading it.** Re-running the 06:00Z window reported `deleted: 144`,
**not 145** — the marker survived the delete. The marker kept its id `cue_j8r7keqc` and its
`created_at` of 06:57:06.834Z, so it was updated rather than replaced, and the first ask is preserved.
`failed: 0`, `dropped: 0`, `window_recorded: true`.

## The hole was sampled, not walked, and here is why

Four windows across `04:36:59Z → 10:39:35Z`. Three carry speech, one is quiet at lunchtime. Detail is
in `ETA-SPEECH-TURNS-HOLE-SAMPLE-REPORT-22-AUG-2026.md`.

We stopped rather than walking the rest because the second probe found the defect that ruling still
sits with you: **`stt_silence` cannot fire for actual silence.** `lib/whisper.ts:207-209` returns
`{ ok: false, error: 'empty_transcript' }` on an empty transcript, so the tool aborts before the
silence path. A quiet window writes nothing — no turns and no marker — and the day cannot say it was
asked about. Walking six hours in that state produces gaps we could not distinguish from windows
nobody looked at.

`empty_transcript` fires only on a 200 with no text; a non-200 returns `http_${status}`. Silence is
already distinguishable from a broken call.

## What is actually owed

1. **The silence ruling.** It unblocks the walk.
2. **Five Monday questions**, in `ETA-MONDAY-READINESS-AND-OPEN-RULINGS-22-AUG-2026.md`. V is
   recording live in OPD on **Monday 24 August**. Consent copy and the second microphone are the two
   that cannot be fixed after the fact.
3. **A sixth**, in the same file: V has asked for a live room monitor in Admin Bench and has approved
   a design. It is being built on his call. You are being told, not asked, because he is the operator
   and Monday is in two days — but it touches your no-room-screen rule closely enough that you should
   hear it from us. No clinician sees it. It lives behind the admin cookie.

Nothing needs rebuilding. Two rulings and a recording day are what stand between here and a walked
hole.
