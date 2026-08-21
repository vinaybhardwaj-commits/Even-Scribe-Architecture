# Designer acceptance — fuse slice 3 (warehouse join)
21 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-SLICE-3-21-AUG-2026.md`. Live `main` `1d30546`. Migration `0047`.

## Call

**Slice 3 accepted.** I re-read both live rooms on 19 Aug from this chat:

- OPD 7: still exactly one cue, `cue_95z4avnv`, `at` `04:02:29.995Z`, `created_at` `04:02:31.544Z`, payload `source: kiosk`.
- Cardiology OPD: still three kiosk `consult_mark`s only. No warehouse row on the live days.

43 warehouse events landed on the scratch days. Second load 0 written / 43 existed. That is the join.

`cue.source_ref` plus the partial unique index `(source_ref, type, at) WHERE source = 'warehouse'` is accepted. Warehouse events have no `bench_session`; slice 2's replay key cannot cover them. Keep `pstart` as the type name. `cue.type` remaining CHECK-less is correct (open set); the catch on `consult_start` is why the report exists.

## The three findings, ratified

**1. Warehouse has no room.** Attribution by `doctor_uid` from the tape labels is the only defensible join. Do not invent a room in the warehouse. Do not load all 22 EHRC doctors into a seven-hour OPD 7 window.

This **does** change how §10.1 is read. It does **not** change §9.

- §9 unchanged: post warehouse clocks at their real times onto scratch. Do not auto-start the tape. Do not write the live 19 Aug `room_day`.
- §10.1 restated: a consult-mark window binds a warehouse `pstart` / `pqm_called` only when that clock is attributed to **this room's doctor**. "Contains" is not hospital-wide. Room-to-doctor is tape-side (session / room labels that day). Fuse must not re-query the warehouse for every doctor.

`pstart` without a mark still mints a visit on scratch (official start, strong identity). That is not "auto-start from warehouse" — that sentence remains about the tape. `pqm_called` without a mark may open `called`. `pulse_note` and `dx_event` still do not start a visit. `attribution: inferred` is evidence, not a binder; do not mint identity from inferred `dx_event` alone.

**2. Keep the cue type `pulse_note`.** It is Scribe's name for the later lock. Pulse's object is a prescription document that is a full OPD consult note, not a drug order. Renaming to `prescription` would teach the fuse the wrong thing. Payload may keep `at_source: uploaded_at` / `timestamp`. Addendum overwrite + `prescription_history` is a snapshot caveat for slice 4, not a rename. IPD / KareXpert templates stay out.

**3. Gaps stay.** Do not filter by tape window. `in_tape_window: false` is a fact. 35 of 43 outside every tape window, Ankit's six-hour warehouse silence with tape running, Cardiology's five hours of tape against one billed consult — those are the day report, not errors. V's call stands.

Absent payload keys stay omitted, never null. Null means the warehouse gave a null.

## Corrections you already made

- `HEALTH_PACKAGE` is a sixth diagnostic category. Include it in `dx_event`. I can see one on scratch Cardiology.
- `dx_event.at` is `check_in_time` (order / check-in), not the procedure. Sample 98 minutes later is expected. Fuse prior 4 already treats this as hole evidence, not "they are in the scanner now." Do not wait for `sample_collection_time`.

## Cliffs, parked

- **§11.5 day report after slice 4, not before.** `scribe_day_report` is tape-only today. Marks vs warehouse vs visits vs tape cannot exist until visits exist. Original slice order stands: 4 then 5. Do not build 5 first.
- **`visit.room_day_id` single-valued is slice 6.** Do not widen 0042 in slice 4. This corpus: the two rooms' patient sets did not overlap. If that still holds after fuse, skip slice 6 rather than invent a cross-room patient. Plural `room_day_ids` remains the product shape; schema catches up when we have one.

## Precondition for slice 4 (not a slice 3 fail)

Live Cardiology has three kiosk marks. Scratch Cardiology currently has one (`06:08Z`). Before the fuse writes visits, replay-write every **ended** Cardiology session for 19 Aug onto the same scratch day so the mark set matches. Do not invent tape for `bs_8temrdqh` (mic dead, zero backup). Crash session `bs_j9wgfa33` is ended; if it has marks, they belong on scratch.

## Slice 4 — fuse on scratch — may proceed

Gemini Flash. Advisory lock already there. Empty fuse must not crash `POST /cues`.

Write visits **only** on the scratch `room_day`s (`rd_scratch_qyzghzaf_20260819`, `rd_scratch_bh6jtq4t_20260819`). Live OPD 7 cue count must stay 1. Live Cardiology must stay three kiosk marks.

Priors to put in front of the model, not as switches:

1. Visit key is `individual_uid` + IST date.
2. Bind a mark to a warehouse clock only for this room's doctor.
3. `pstart` may open `in_chair`. `pulse_note` is a later lock. `dx_event` opens a hole, not a new visit, when there is no second `pstart`.
4. `in_tape_window: false` does not drop the visit and does not attach tape.
5. Voice is a prior, not a binder. 0.78 banned.
6. No kiosk `individual_uid`. No Pulse writes. No Slack. No Gerrit.

Send the slice 4 report here before slice 5, and before any pass that writes the live `room_day`.
