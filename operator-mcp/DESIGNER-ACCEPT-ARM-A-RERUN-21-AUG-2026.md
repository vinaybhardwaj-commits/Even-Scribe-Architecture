# Designer acceptance — arm A re-run
21 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-ARM-A-RERUN-21-AUG-2026.md`. Live `main` `a7105ad`. Migrations `0048` and `0049`. Confirmed from this chat: live OPD 7 still `cue_95z4avnv`; live Cardiology still three kiosk marks and an empty live visit list. Scratch Cardiology 3 visits. Scratch OPD 7 13 visits, all ended, `active_visit_id` null. No individual ids in this note.

## Call

**Arm A re-run accepted. Slice 5 may proceed.** You were right about 13. I predicted 14 without the 04:31 / 04:36 clocks. The mark window was not empty, so the mark must not also mint `unknown`. The six-hour gap was never that visit. It is `04:36:59Z → 10:39:35Z` on the cue timeline, and slice 5 shows it from there.

`LAST_MARK_WINDOW_MS` = 45 minutes stays a named prior, on the scoreboard, not buried.

## Answers

**1. One visit per person, not one per mark.** Two different `individual_uid`s in one mark window are two visits. The mark is consumed (no extra `unknown` row). Collapsing them into one visit would drop a person. Same `individual_uid` with `pqm_called` then `pstart` in the same window is **one** visit (`pstart` is the stronger opener). Your two-clock / two-visit result is correct if those two calls are two people. The sentence you quoted meant "do not also mint the mark-only row." It did not mean one visit per tap.

**2. Accept `end_reason = 'day_rollover_at_diagnostics'`.** Open set, no new column. A visit that went `in_chair → at_diagnostics` and then hit the day boundary is not the same as one that never closed. Slice 5 counts the token. The `dx_event` cue remains the independent check. `unknown` rows stay `unknown` — not closed by `pulse_note` or `day_rollover`. That is how extra Cardiology taps survive.

**3. Confirm the pulse_note target, with one cut.** For that `individual_uid`: the latest visit already open at the note's timestamp (`opened_at ≤ note.at`). Else the latest still-open visit. Else **do nothing** — do not close "the latest of any" if it is already `ended`. The note does not mint, does not reopen, does not overwrite an existing `end_reason`.

## Slice 5

Marks vs warehouse vs visits vs tape. Name `arm: rules`.

Must show:

- Cardiology's two extra marks as `unknown` 0.3
- Cardiology's one billed consult bound in the 06:08 window, closed on `pulse_note`
- OPD 7's six-hour cue-timeline gap (`04:36:59Z → 10:39:35Z`), not a visit row
- Nine afternoon `pstart` visits outside the last-mark window
- `LAST_MARK_WINDOW_MS` as a named parameter
- Closed-set `ambiguity` tokens, split on comma

No live `room_day`. No B. No C.
