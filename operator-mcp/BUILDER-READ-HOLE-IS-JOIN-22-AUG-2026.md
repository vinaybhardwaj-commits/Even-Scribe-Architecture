# The read: the six-hour hole is our join, not a Pulse outage

22 August 2026. From the orchestrator, to the Scribe Designer.
Read-only, Metabase db13, before Monday's first patient. No Pulse write. No Gerrit.

**You were right. The blackout is our filter.** Fifty-two doctors were working at Even during the
window we called silent. We bound OPD 7 to one `doctor_uid` and dropped the other fifty-one.

Worse than that: **there is no room dimension in the warehouse at all**, so binding a room to a doctor
was never a join. It was a guess that happens to work only when a room hosts exactly one doctor all
day.

---

## 1. What the window actually holds

`19 Aug 2026, 04:36:59Z → 10:39:35Z`, all doctors, `dpipe_prescription_pipeline`, matching on either
`doctor_start_time` or `uploaded_at`:

**52 doctors. Hundreds of prescriptions.** The busiest single clinician has 23 in that window.

The query independently reproduces a fact we already hold, which is the check that it is reading the
right thing: **Dibyendu returns exactly 1.** Your standing "Dibyendu = 1" survives contact with the
unfiltered table.

## 2. The dermatology consult has candidates

The hole walk found a dermatology consultation with a full vitiligo prescription at **11:30–11:36 IST**
(`06:00–06:06Z`).

That morning there were **five dermatologists** working. One of them has a first prescription start of
**11:29:36 IST** — twenty-four seconds before that window opens.

We are not asserting who was in the room. We are saying the room was not empty, the warehouse knew
somebody was consulting, and our filter threw it away.

## 3. The structural finding: the warehouse has no room

This is the part that matters beyond 19 August.

- `dpipe_prescription_pipeline` has **no room column and no hospital column**. Doctor, times, clinical
  content, and nothing about where.
- `dpipe_pqm_tokens` has `hospital_uid` and `hospital_name` but **no room**.
- `even_hospitals` has a column named `doctor_opd_rooms`. It is `jsonb`. **Three hospitals, zero with
  a non-null value.** The one field that could carry a doctor-to-room mapping exists and has never
  been populated.

Your point 3 is confirmed and then some. There is no room to be missing — there is no room at all.

## 4. What this does to the fuse picture of 19 August

The scratch day's warehouse timeline was loaded from a one-doctor extract. So every warehouse number
we have reported for OPD 7 is **one doctor's**, not the room's:

- `warehouse_total: 39` is one clinician's 39.
- "12 of 39 fall inside the tape" is one clinician's 12.
- The `whole_day` and `between` silences were measured against one clinician's clocks.

The hole walk stands — the tape is the tape, and 1,708 segments were recovered. **What does not stand
is the claim that the warehouse recorded nothing.** It recorded plenty. We did not load it.

## 5. What this means for Monday

**Monday will reproduce the hole if we join the same way**, and we would misdiagnose it a second time.

Two paths, and they are not equivalent:

**a. Stop filtering by doctor.** Load every doctor's clocks for the day and accept that the warehouse
cannot say which room. The tape then becomes the only thing that knows the room, and a cue is evidence
that *somebody* was consulting somewhere at that instant. Honest, immediately available, and it makes
`marks_unaccounted` mean something different — it would need re-reading.

**b. Populate `doctor_opd_rooms`.** Then a real join exists. That is a product change, not an analysis
one, and it is not a weekend job.

We recommend (a) for Monday and (b) as the actual fix. We have changed nothing and will not until you
rule.

## 6. What we did not touch

No Pulse write. No Gerrit. No live `room_day`. No re-walk. The scratch graph is unchanged — this was
four read-only queries against Metabase, exactly the tables the loader's extract came from.

## 7. What we ask

1. **Rule on the Monday join.** (a), (b), or something else.
2. **Say what happens to the 19 August warehouse numbers.** They are a one-doctor slice presented as a
   room. Re-load unfiltered, or annotate and leave?
3. `marks_unaccounted` and the silence edges were computed against that slice. If we re-load, they
   change. Say whether the fuse re-runs or the report carries a correction.

V is briefing the doctors himself and is handling consent and the microphones. This read was the one
item on the list that was ours.
