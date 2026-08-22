# Designer acceptance — slice 5, the scoreboard
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-SLICE-5-21-AUG-2026.md`. Live `main` `fcea93e`. I ran `scribe_fuse_report` myself on both scratch days and on live OPD 7. Live OPD 7 is still `cue_95z4avnv` only; live Cardiology still three kiosk marks and zero visits. No individual ids in this note.

## Call

**Slice 5 accepted.** The two required numbers are on the tool:

- OPD 7 hole: `04:36:59Z → 10:39:35Z`, 6h 2m 36s, `edge: between`, `tape_running: true`. No visit represents it.
- Cardiology `marks_unaccounted: 2`, both `unknown` at 0.3.

Silence now has edges. Cardiology's first run saying `silence: []` was a bug; leading 2h 14m and trailing 3h 20m are now visible. Live OPD 7 reports `whole_day`. Negative intervals are not silences. `bs_j9wgfa33` `end_time_disagrees` is on the scoreboard, not in a note. Parameters are named, including `LAST_MARK_WINDOW_MS` = 45 min.

The 19 Aug finding is now measured: tape and warehouse barely overlap. That is the artefact.

## Answers

**1. Do not just rename. Split the count.** `warehouse_bound_to_visit: 13` / `unbound: 26` on OPD 7 hides 12 notes that closed a visit and 4 `dx_event`s that moved one. Cardiology's `1 / 3` is the same lie about one consult. Four fields:

- `warehouse_opened_a_visit`
- `warehouse_closed_a_visit`
- `warehouse_moved_a_visit`
- `warehouse_unbound` — only events that did none of the three

Small follow-up on the same tool. Not a new slice. Not a reason to hold this acceptance.

**2. Do not move `LAST_MARK_WINDOW_MS` on this corpus.** 45 minutes stays the named prior. A final tap whose Pulse clock lands later will look unaccounted; that is a second-day question, not a constant we slide tonight.

**3. JSON behind the MCP door is the artefact.** No clinician page. No room screen. If you render later, admin Bench only, next to `timeline.md`. Do not build that unless I ask.

## Programme

§11 slices 1–5 are done. Slice 6 stays skipped: the two rooms' patient sets did not overlap, and I will not invent a cross-room patient.

**Do not write the live `room_day`.** I have seen the scoreboard. Promotion is still a separate write, designed on purpose. Live OPD 7 today is one mark and a `whole_day` warehouse silence. Scratch is a reconstructed warehouse-shaped day. Those are two pictures. Do not merge them because the programme finished.

No B. No C. Default arm stays `rules`.
