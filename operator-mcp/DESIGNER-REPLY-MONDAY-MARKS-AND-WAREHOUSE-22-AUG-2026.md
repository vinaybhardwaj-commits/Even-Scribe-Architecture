# Designer reply — two Monday rulings
22 Aug 2026. Scribe Designer. For V and the orchestrator.

Marks: one tap in seven hours on OPD 7, 19 Aug. Warehouse: a six-hour blackout that the tape says was a morning clinic. Monday is 24 Aug.

## 1. Marks — brief the doctors. Do not restyle the kiosk. Operator mark is fallback only.

The lever for Monday is **verbal briefing**, not a new button and not the admin monitor.

The kiosk already has Mark consult. They pressed it once. A weekend restyle will not teach a habit. Leave the control as it is. Do not add a second mark affordance on the kiosk this weekend.

**Operator marking from the admin monitor is not the plan.** That page stays observe-first. `scribe_mark_consult` remains a fallback when V can hear the room and says why. It is not how we get a day's marks. Using it as the standing lever makes the operator the second fuse we forbade, and a mark from outside the door will be late.

Briefing, in the room, before the first patient: the day is already recording; Mark consult is a tap when a consult starts, not Pulse start-prescription, not stop. If they forget, V may mark from admin with a reason. That is the exception.

Consent copy is still counsel's. I do not write it.

## 2. Warehouse — yes, read-only, before Monday.

The blackout may be Pulse not used. It may also be **our join**. Warehouse has no room. We bound OPD 7 to Ankit's `doctor_uid`. The hole walk found a dermatology consult with a full prescription in that room. If another doctor sat there, every one of their clocks was dropped by the filter, and Monday will "reproduce" a hole that is a join bug.

Read only. No Pulse write. No Gerrit. Metabase first (same tables as the loader). Pulse schema only if Metabase cannot answer.

Look at 19 Aug IST, `04:36:59Z → 10:39:35Z`, before inventing a Pulse outage:

1. **Other doctors.** Any `pstart` / `pqm_called` / `pulse_note` / `dx_event` in that window for any EHRC doctor, not only Ankit. If a dermatologist (or anyone else) has a note or start that morning, the hole is a filter, not a blackout.
2. **Ankit himself.** Did he have queue or starts that morning that we already loaded as `in_tape_window: false`, or truly nothing until 10:39Z.
3. **Room still missing.** Confirm `assigned_bay` / routing still has no OPD 7. Do not invent a room in the warehouse.

Send that read here before Monday's first patient. If it is a filter, Monday's join must not assume the tape label is the only doctor in the room. If it is truly empty for every doctor, Monday may reproduce a Pulse-side hole, and the tape is the record.

Do not slide `LAST_MARK_WINDOW_MS`. Do not write the live `room_day`. Do not re-run B or C.
