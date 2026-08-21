# Designer acceptance — slice 4 bake-off
21 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-SLICE-4-BAKEOFF-21-AUG-2026.md`. Live `main` `154205b`. Migration `0048`. Confirmed from this chat: live OPD 7 still `cue_95z4avnv`; live Cardiology still three kiosk marks. I read the scratch pictures. 18 `arm:'rules'` rows. No individual ids in this note.

## Call

**Arm A wins the bake-off. Arms B and C are disqualified. Arm A is not yet the slice 5 picture.**

C deleted three clinician marks on Cardiology and reported a clean one-visit day. That erases the finding we ratified this morning. B kept the rows and set confidence to 0 — a soft version of the same erasure. A kept `unknown` at 0.3, twice, on both rooms. On the criterion I wrote, A is the only arm that can be accepted.

Opening-evidence `(arm, opened_by)` is why identity was stable. That key stays.

Do not write B or C into `visit`. Do not add a run id to `visit` so the union of a non-deterministic arm can be scored as a result. This report is the scoreboard for B and C. Default read stays `rules`.

## Answers

**1. Winner — A.** Determinism, and it refuses to delete a consultation. B and C are done for this corpus. Do not re-run them unless I ask. The product fuse is still a model for the cases I named yesterday (dx return by voice, two open visits, STT roles, pin reject, cross-room). Those are not this corpus. §7 is unchanged.

**2. Does `pulse_note` close a visit? Yes, that individual's open visit.** Later lock, not a start. `end_reason='pulse_note'`. It does not mint a visit. It does not close anyone else. Mark-only unknowns have no `individual_uid`, so a note cannot close them.

After the finished day's cues are fused, nothing should still sit `in_chair`. Close on `pulse_note`, then `day_rollover` anything still open. A closed room-day has `active_visit_id` null. Right now scratch OPD 7 still has a dozen `in_chair` rows because A never closed; that is the bug this answer fixes.

**3. Reload `calendar_uid` before slice 5. Yes.** It is on every warehouse `pstart` and never reached the cue. §10.3 is untestable until it does. Add it to the payload, delete warehouse cues on the **scratch** days only, reload, re-run **A only**. Live days stay untouched. A second `pstart` + new `calendar_uid` is a second visit. A second `pstart` with the same `calendar_uid` is not.

**4. Do not store B and C as visit rows.** The dry-run report is enough. A run id on `visit` would accumulate C's 14-then-13 into a ghost set. If you want the dry-run JSON next to this report in the architecture repo, fine. Not Neon.

## Fix A before slice 5 — the invented 30 minutes

`MARK_BIND_WINDOW_MS = 30 min` is not in the PRD. It split Cardiology's one billed consult into two visits: the 06:08 mark stays `unknown`, and the 06:54 `pstart` (46 minutes later) opens a fourth row. I can see that on scratch. Same class of error as C, milder (both rows survive).

§10.1's "consult-mark window" is not ±30 minutes around a tap.

**Named prior, v1:**

- If a later mark exists on that room-day, the window is `[this mark, next mark)`.
- If this is the last mark of the day, the window is `[this mark, this mark + 45 min)`. 45 minutes is a named prior (Pulse often opens after talking). Put it in the report. Do not hide a constant.
- A `pstart` / `pqm_called` in the window binds to that mark: one visit, identity from the warehouse clock, not a second row.
- A warehouse clock outside every mark window still mints its own visit (`pstart` without a mark). That is Ankit's afternoon.
- A mark with an empty window stays `unknown` at 0.3. That is the two extra Cardiology taps.

On this corpus that should make Cardiology **3 visits, not 4**. OPD 7's single morning mark must not swallow the 10:39Z pstarts. The six-hour gap stays a gap.

## `end_reason` is not for doubt

`state: in_chair, end_reason: pstart_without_calendar_uid` is the field doing two jobs. Closed-set reasons (`pstart_without_calendar_uid`, `mark_without_warehouse_evidence`, `inferred_attribution_only`, `pqm_called_without_pstart`) belong on a nullable `visit.ambiguity`. `end_reason` is only set when `state='ended'`. Small additive (0049) on the same A re-run is fine. Do not invent prose.

## Hard locks that held

Fail-closed on provider: B and C reported `gemini:…` and never wrote. Keep that. B ran `gemini-3.1-pro-preview`; I asked for 3.7 Flash on B and C. It does not change the disqualification. Do not pull Pro for a later model arm.

`scribe_llm_health` existing is good. Two months of notes on `qwen2.5:14b` is not this slice; do not expand it here.

## Next

1. Payload `calendar_uid` on warehouse `pstart`. Scratch delete + reload. Live cue counts unchanged.
2. Bind rule as above. Close on `pulse_note`, then `day_rollover`. `ambiguity` not `end_reason`.
3. Re-run **arm A only** on the two scratch days. Idempotent on `(arm, opened_by)`.
4. Slice 5 is the scoreboard, and it names `arm: rules`. Marks vs warehouse vs visits vs tape. Cardiology must still show the extra marks. OPD 7 must still show the six-hour gap.

Send that A re-run + slice 5 together. No live `room_day`. No B. No C.
