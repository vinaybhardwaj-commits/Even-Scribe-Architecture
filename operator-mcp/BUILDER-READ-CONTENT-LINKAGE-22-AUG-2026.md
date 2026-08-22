# Content linkage works, and it explains the hole for the third time

22 August 2026. From the orchestrator, to the Scribe Designer.
Read-only Metabase db13 plus one dry-run transcription. Nothing written.

V proposed linking tape conversations to prescriptions by content, then using voice to attribute the
doctor. **The content half is proved. The voice half turns out not to be needed for it.** And the same
query settles what the six-hour hole actually was — for the third and final time.

---

## 1. The proved pair

**Tape**, OPD 7, `10:37:00Z–10:40:00Z` (16:07–16:10 IST): a dog-bite history. Mechanism, timing, the
animal's vaccination status, its breed and age, and the bite site. Six turns, read directly rather
than summarised.

**Prescription**, Metabase, uploaded **16:18:50 IST**, doctor **Ankit**, diagnosis
**`W54.0XXA` — Dog bite**.

Eight minutes between the consultation on the tape and the prescription in Pulse. Same doctor, same
room label, a distinctive presentation.

**This is the linkage, and it needed no voice.** Content plus time did it.

## 2. The hole, explained a third time

We have now had three explanations and this one is correct.

| explanation | verdict |
|---|---|
| Pulse was down for six hours | **false** — 52 doctors, 516 prescriptions that day |
| We filtered to one doctor and dropped 51 | **true, but not the reason the room looked empty** |
| **The bound doctor was not in the room** | **this is it** |

Ankit's 10 prescriptions on 19 August: one at 09:15 IST, then **nothing until 16:18**, then nine
between 16:18 and 17:51. **His clinic was the late afternoon.** The tape ran 09:31–16:42, so it
captured six hours of somebody else's clinic and the first twenty-four minutes of his.

His 09:15:49 IST upload is `03:45:49Z` — the first warehouse event on the scratch day, sixteen minutes
before the tape started. That confirms he is the doctor the loader bound to OPD 7.

So the room was never empty and Pulse was never down. **We attributed a room to a doctor who had not
arrived yet.** That is the same defect as the missing room dimension, seen from the other end.

## 3. What content matching can and cannot do

**It works**, on the evidence of §1.

**It is not a universal key**, on the evidence of two independent limits measured today:

- **36% of prescriptions carry no medicines at all.** 188 of 516 on 19 August have `medicines` null.
  A drug-name match fails on a third of the corpus. The dog-bite pair matched on *diagnosis*, not
  drugs, which is the more robust signal.
- **Some care has no prescription at all.** The vitiligo consultation in the hole — a full regimen
  dictated aloud, with drug, strength, dosing days, a topical, a lab check and a review interval — has
  **no matching row anywhere in the 516**. Zero vitiligo diagnoses, zero levamisole, zero tofacitinib,
  drafts included.

## 4. The reframe

V reached for this as an **attribution** mechanism. The data says it is better as **two** things, and
the second is worth more:

**a. A candidate producer.** Exactly the shape you ruled for Monday: when the tape carries clinical
content and the labelled doctor has no clock, content matching proposes *candidates* against the
hospital-wide extract. Attribution stays `inferred`. Never bound. The dog-bite pair is what a
high-confidence candidate looks like.

**b. A detector of unrecorded care.** A tape window carrying a coherent clinical conversation with no
matching prescription anywhere is a finding in its own right. The vitiligo consultation is one. Care
was delivered and prescribed aloud, and the record does not have it.

(b) does not need voice, does not need `speaker_cluster`, and does not need slice B.

## 5. What this does to slice B

Voice was going to carry attribution. It now carries less than we thought, and it is further away than
V assumed:

- **Only two enrolled voiceprints exist in the whole system** — Darshan Gowda and V. Neither Ankit nor
  Dibyendu has one, so there is no reference centroid for either doctor in these rooms.
- The 17 August far-field probe measured **0.55–0.62** for a known speaker in a room against a
  phone-enrolled centroid, versus a live threshold of **0.78**. It works as evidence at a recalibrated
  0.45–0.50, never as a verdict.
- **Impostor separation has never been measured.** Without it there is no safe threshold.

Content reaches further, sooner, with no enrolment and no new stack.

## 6. What we ask

1. **Rule on content matching as the candidate producer**, in place of voice, for the mechanism you
   already specified in the Monday join.
2. **Rule on unrecorded-care detection** as a finding the scoreboard may carry — or explicitly not, if
   you want it kept out until the corpus is bigger.
3. **Slice B's purpose changes** if you accept (1). Voice becomes a later refinement rather than the
   attribution mechanism. Worth restating before the brief is written.

Nothing was written. No live `room_day`. No Pulse write. No patient detail appears in this document.
