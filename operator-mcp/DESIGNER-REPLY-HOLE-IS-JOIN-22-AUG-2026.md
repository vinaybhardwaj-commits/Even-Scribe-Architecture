# Designer reply — the hole is our join
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-READ-HOLE-IS-JOIN-22-AUG-2026.md`. Read-only Metabase, no Pulse write. Dibyendu = 1 still holds on the unfiltered table.

## Call

**Accepted.** The six-hour hole is our filter, not a Pulse outage. Fifty-two doctors were working. We bound OPD 7 to one `doctor_uid` and dropped the rest. The warehouse still has no room. `doctor_opd_rooms` exists and is empty. Binding a room to a doctor was a guess that only works when one doctor owns the room all day.

The hole walk still stands. The tape is the tape. What does not stand is "the warehouse recorded nothing."

## 1. Monday join — neither (a) nor (b) as they are written. Do (c).

**(a) is refused as a room load.** I already forbade loading the hospital into one OPD 7 window (slice 3, twenty-two doctors then, fifty-two now). Unfiltered clocks written onto the room scratch would mint visits for people who were never in that room. `marks_unaccounted` would become noise. That is a tidier lie in the other direction.

**(b) is the real fix and not a Monday job.** Populating `even_hospitals.doctor_opd_rooms` is a Pulse product change. We do not write Pulse. We do not do it this weekend. Recommend it to Pulse after Monday. Until that field is real, there is no warehouse join to a room.

**(c) Monday:**

1. **Room scratch stays doctor-scoped** to the tape label (the doctor the kiosk says is in the room). Same loader as now. Same visit writer (arm A). Same "do not invent a room."
2. **Hospital-day extract is a lookup, not a room cue.** The query you just ran — every doctor, that IST day or that window — lives as a report / side extract. It is not inserted onto `rd_scratch_*` as warehouse cues for that room.
3. **Mismatch is a finding, not a mint.** If the tape has speech (or a mark) in a window where the labelled doctor has no clock, and the hospital extract has clocks in that window, report candidates. Attribution stays `inferred`. Do not bind. Do not mint a visit for each candidate. The dermatology window plus a first start at 11:29:36 IST is exactly that shape: a candidate, not a name on the visit.
4. **Do not assume the tape label is the only doctor in the room.** Say so on the scoreboard. The labelled doctor's silence is not the hospital's silence.

That is available Monday. It does not require a Pulse write. It does not require a fuse rewrite.

## 2. 19 August warehouse numbers — annotate and leave.

Do not re-load fifty-two doctors onto `rd_scratch_qyzghzaf_20260819`. Those 39 cues are Ankit's extract. Relabelling them as "the room" was the error. Keep the rows. Name them **doctor-scoped** on the next report.

The 1,708 segments stay. Live OPD 7 stays one cue.

## 3. Fuse — do not re-run. The report carries the correction.

Re-running A against an unfiltered load would invent a hospital-shaped OPD 7. Do not do it. Do not re-run B or C.

Correction, named, on the scoreboard / close note:

- `warehouse_total: 39` is one clinician, not the room.
- "12 of 39 inside the tape" is that clinician's 12.
- The `between` / `whole_day` silences were measured against that clinician.
- Hospital-wide, the hole window had 52 doctors and hundreds of prescriptions.
- One dermatologist's first start is 24 seconds before the 11:30 IST probe. Candidate, not a bind.

`marks_unaccounted` stays meaningful against **marks vs the labelled doctor's clocks**. It is not "the warehouse had no care."

## Unchanged

No Pulse write. No Gerrit. No live `room_day`. No visit minted from STT. Window-as-unit stays. Briefing is still the Monday mark lever. Operator mark is still fallback only. I still do not write consent copy.
