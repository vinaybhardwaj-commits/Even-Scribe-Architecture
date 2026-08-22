# Designer reply — OPD 7 hole, walked
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-OPD7-HOLE-WALK-22-AUG-2026.md`. Prod `8820b5c`, migrations through `0053`.

## Call

**K5 accepted.** Silence records itself. The 07:45Z window that wrote nothing this morning now has `stt_silence` and `stt_window.complete: true`. Building the silent turn-set from zero segments is better than the spec. Keep that.

**The walk is accepted as the picture of the hole.** Twenty-five contiguous windows, every one complete, zero failed. Three phases: a working morning clinic (~10:07–12:22 IST) with real consultations and no warehouse event; a dead lunch (~12:22–14:07); an afternoon whose counts look busy and are mostly not.

**OPD 7 on 19 Aug is heard**, subject to the two rulings below. That does not mint a visit. Arm A stays the visit writer. This is evidence Pulse missed a morning of care, not a new scratch graph of patients.

Do not write the live `room_day`. Live OPD 7 stays one cue.

## 1. Hallucinated silence — do not convert it to `stt_silence`

`stt_silence` stays what K5 made it: Whisper returned empty. One in eight quiet windows is the ear being honest, not a coverage target we paper over.

Do not add VAD. Do not add a second STT stack. Do not invent a content threshold that rewrites filler as silence. A heuristic that mints silence will hide the same defect the bake-off hid: a tidier day than the one that happened.

Named prior, report-only, no re-walk:

- `segment_count = 0` → silence (already).
- `segment_count` 1–3 on a window ≥ 10 minutes → **thin**. Not a consult. Not silence.
- A window the walk already called a loop (identical text repeated) stays speech in the graph and is **loop** in the report. Do not delete those turns this weekend.

Downstream must not treat thin or loop as a visit. Do not add a scoreboard field for this walk. The table in your report is the artefact.

A hallucination classifier is a later slice, after Monday, if we still want one. Not a Monday build.

## 2. Non-overlapping windows — ratify. Delete the probe overlap.

Two overlapping asks are two asks. A different window is a different clip and a different segmentation. That is why `already_existed` was 0. Walks are contiguous and non-overlapping. The window is the write unit.

The four earlier probes are scaffolding. The contiguous walk is the day. **Delete the overlapping probe turns** (the four asked windows that sit inside `04:36:59Z → 10:39:35Z` and are not on the 15-minute grid). Use the existing window delete. Do not add a tool. Leave the 25 walk windows and the 04:02Z proof window.

After that delete, counts on the hole are the walk. The probes stay in the written report as how we found the finding, not as a second listen.

## 3. Cardiology — after Monday is safe, not as a Monday gate.

Do not walk Cardiology's finished sessions before Monday if that competes with consent, the second microphone, or the admin monitor. Those three cannot be fixed after the day starts.

If Monday prep is already done and Mini is idle, a Cardiology walk is allowed. It is not required to call 19 Aug heard. **OPD 7 is heard. Cardiology is not.** Say so on the next spin-up.

I still do not have `ETA-MONDAY-READINESS-AND-OPEN-RULINGS-22-AUG-2026.md`. Send it. I will not guess the five questions. I still do not write consent copy.

## Unchanged

No live `room_day`. No B or C. No run id on `visit`. No transcripts in Slack, chat, or this repo. `stt_turn` does not mint or close a visit. Window-as-unit stays. Marker stays INSERT-only. Dibyendu = 1. No Pulse. No Gerrit.
