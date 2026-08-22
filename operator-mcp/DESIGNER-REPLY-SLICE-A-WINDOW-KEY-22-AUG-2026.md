# Designer reply — slice A, the window is the write
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-SLICE-A-WINDOWS-22-AUG-2026.md`. Prod `ded0451`, migrations through `0051`.

## Call

**The first listen is accepted as a proof, not as a corpus.** We can hear Dibyendu's consult. The client, the named conflict target, `0051`, the stitch, the scratch guard, and the dry default all held. Ruling 2 as a *cross-run* key does not. Whisper is not a deterministic writer. I am changing the write unit, not the turn shape.

Do not walk the rest of the day. Do not analyse the 77 Cardiology turn rows. Do the four things below, then the hole window, then one complete Cardiology window. Then I read.

## Answers

**1. Window-as-unit. Ruling 2 changes for cross-run writes.**

A turn is still one cue, `at` = start, `payload.end` = end. `source_ref` is still `{session_id}|{start_ms}|{end_ms}|{speaker}` and still the *within-write* key (two turns that share a start still both land — `0051` proved that, keep it). It is **not** the idempotency key across runs. Segmentation is Whisper's opinion about a window. An opinion is replaced, not merged.

Write unit: `(session_id, asked_window_start, asked_window_end)` on that scratch `room_day`.

In one transaction:

1. Delete existing `stt_turn` and `stt_silence` for that pair (match `session_id` and the asked window on the payload, not Whisper's segment times).
2. Insert the new set.

Report `deleted / written / already existed / dropped`. A second run of the same window may write a different segment count. That is a replace, not a fail. `already existed` should be 0 after a delete. A drop that is not a named within-write conflict is still a bug.

Do not take a new `CUE_SOURCES` value. Source stays `replay`.

Pin the decoder as you recommended: greedy, beam 1, temperature 0, fixed seed, if the Mini whisper.cpp endpoint accepts those params. Second, not instead. It will not make the writer deterministic. Do not invent a second STT stack.

**2. Refuse a window you cannot finish. Also record that you asked.**

A silent partial is worse than a refusal. `stopped_early` in the MCP response and nowhere on the day is the defect.

- Raise the budget so a six-minute stitched window can finish in one transaction. The 30-minute ask cap stays.
- If you still cannot finish: **do not commit turn rows.** Roll back the turn insert. Do not leave 71 of 162.
- Write (or replace) one completeness cue:

```
type        stt_window
source      replay
at          asked window start
source_ref  {session_id}|{window_start_ms}|{window_end_ms}|window
payload     { end, engine, source_used, complete, segment_count, language, stopped_early? }
```

`complete: false` with `stopped_early: time_budget` and `segment_count` of what Whisper returned, zero `stt_turn` rows for that window. A reader can tell "we asked and failed" from "we never asked."

`complete: true` rides in the same transaction as the turns (or the `stt_silence`). Delete-then-insert applies to `stt_window` for that pair too.

Add `stt_window` to the `scribe_post_cue` blocklist and to the `0050` / `cue_turn_natural_key` type lists (same family as `stt_turn` / `stt_silence`).

Do not write a partial turn set "to get something on the day."

**3. Cleanup — confirm.**

Delete the 77 Cardiology `stt_turn` rows on `rd_scratch_bh6jtq4t_20260819` before the fixed writer runs. They are two disagreeing opinions of an unfinished window. Evidence of the defect, not a picture of the consult.

OPD 7's 35 rows at 04:02Z may stay. That window is complete and stable. It is not the hole. Do not treat it as the hole.

**4. Then the hole, then a complete Cardiology window.**

Order:

1. Ship the window-unit writer, the decoder pin, `stt_window`, the refuse-partial rule.
2. Delete the 77 Cardiology turns.
3. One window inside OPD 7 `04:36:59Z → 10:39:35Z`. Words or `stt_silence`. No new visit. This is still the product question.
4. Re-run Cardiology around 06:54Z as one complete window (`stt_window.complete: true`). Do not analyse the old 77.

Send that two-window report before walking the rest of the day. Same dry default. Same scratch guard. Live OPD 7 stays one cue. Live Cardiology stays three kiosk marks.

## Unchanged

One cue per turn. `payload.end`. Source `replay`. Blocklist first. Asked window, not the slab. One language per file. Dibyendu = 1. No live `room_day`. No Pulse. No Slack. No Gerrit. `stt_turn` does not mint or close a visit. Arm A stays the visit writer. No transcripts in Slack, chat, or this repo.

The 35-of-35 OPD 7 single-chunk pass still stands as proof that a deterministic window is replace-safe. The messy window is the one that matters, and it is why the write unit is the window.
