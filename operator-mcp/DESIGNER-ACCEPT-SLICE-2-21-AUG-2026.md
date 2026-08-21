# Designer acceptance — fuse slice 2 (scratch graph)
21 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-SLICE-2-21-AUG-2026.md`. Live `main` `def3a8e` (slice 2 `614d0c2` + hardening).

## Call

**Slice 2 accepted.** I re-read OPD 7 on 19 Aug from this chat: still exactly one cue, `cue_95z4avnv`, `at` `04:02:29.995Z`, `created_at` `04:02:31.544Z`, payload `source: kiosk`. The live graph was not written.

Migration 0046 (scratch flag, cue `session_id` + `source` columns, partial unique index on replay) is accepted as the first schema change of this programme.

## The two shape changes (ratified)

Both were forced by the tables. Intent held.

1. **Scratch hangs off its own scratch room.** `UNIQUE (room_id, ist_date)` cannot hold two days for the same real room. A flag is not in that key. Do not widen the live conflict target. Hide the scratch room from `scribe_list_rooms`. The write guard still refuses a day whose `scratch` flag is not true, inside the lock.

2. **Promotion is a separate write, not a flip.** When I say the live `room_day` may be written, you design that pass on purpose. Do not `UPDATE room_day SET scratch=false` on the scratch room's day. Do not merge scratch cues onto the live OPD 7 row by changing `room_id`.

## Answers you asked for

**1. `payload.source` stays the original provenance (`kiosk`, `warehouse`, `mcp`).** Do not force `replay` into the payload. The **column** `source='replay'` is what the index keys on and what slice 4 reads. Replay must stay faithful. This does **not** go into slice 3.

**2. The scratch room does not disturb fuse §9 or §11** if these hold:

- Visit key remains `individual_uid` + IST date. Scratch room is not a second real room.
- Live `GET /state` / `scribe_list_cues` / `scribe_day_report` on OPD 7 / Cardiology never include scratch-room cues.
- Fuse slice 4 reads the scratch `room_day` (by id or by the hidden room) and writes visits only on that scratch day.
- `scribe_replay_write` still refuses a session that is not ended. Dry-run stays a separate read-scope tool.

§9 “second pass may write the live room_day” is deferred until I have seen a scratch day report with warehouse + fuse. Not this week’s default.

## Next

**Slice 3 — warehouse join — may proceed.** Post `pqm_called` / `pstart` / `dx_event` / `pulse_note` **only** onto the scratch day. Natural key still session + type + at (or warehouse id + at if there is no session). Live 19 Aug cue count on OPD 7 must stay 1.

Then slice 4 (fuse on scratch) and slice 5 (marks vs warehouse vs visits vs tape).

Send the slice 3 report here before slice 4 writes visits.
