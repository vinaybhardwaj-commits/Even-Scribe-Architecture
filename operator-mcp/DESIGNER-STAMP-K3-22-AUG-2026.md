# Designer stamp — K3 kickoff
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-KICKOFF-K3-WINDOW-UNIT-22-AUG-2026.md`. This is the kickoff that produced the two-window report. **Do not rebuild K3.** Prod moved on: two-window report is `c93078f`, migrations through `0053`.

## What in it still stands

Window-as-unit. Batch insert on the existing cues route. Refuse a partial turn set. Decoder pin. `0051` within-write key. Source `replay`. Scratch guard and lock reused, not duplicated. OPD 7's 35 rows at 04:02Z left alone. No live `room_day`. Dry default. No second STT stack.

The 77 Cardiology rows are already gone. The writer's own delete swept them.

## What this file must not be used for

**§1 and §3 put `stt_window` through the same DELETE as the turns.** That is the bug the two-window report found. The fallback marker died on the same missing grant, and the day stayed silent.

Superseded by `DESIGNER-REPLY-SLICE-A-TWO-WINDOWS-22-AUG-2026.md`:

- `stt_window` is INSERT / `ON CONFLICT DO UPDATE` only. No `replace_window`. No DELETE.
- Counts are `deleted / written / already_existed / dropped / failed`. `dropped` is only a named within-write conflict.
- DELETE on `cue` (`0053`) is accepted. Do not DELETE `visit` or `speaker_cluster`.
- Next: ship the marker upsert and the count split, then walk the rest of the OPD 7 hole.

A kickoff that still says "delete-then-insert applies to `stt_window` exactly as for turns" is not the next build.
