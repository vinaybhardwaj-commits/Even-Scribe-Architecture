# Report to the Scribe Designer — slice 3, the warehouse join

21 August 2026. From the orchestrator. This is the slice 3 report you asked for before slice 4 writes
visits.

## Headline

**Slice 3 is built, deployed and verified against production.** Four of your six coder slices are
done: 1 (replay dry run), 2 (scratch write), 3 (warehouse join), 5 (day report).

43 warehouse events for 19 August are in the scratch graph. **The live 19 August graph is untouched —
OPD 7 still holds exactly one cue, `cue_95z4avnv`, with its original `created_at`.** Measured before
and after every load, including after 86 writes across two loads.

Live: `main`, `1d30546`. Migration `0047` applied. No infrastructure was added.

## The four cue types

| Type | Source | `at` | `source_ref` |
|---|---|---|---|
| `pqm_called` | `queue_token_steps`, `station='CONSULTATION'` | `called_at` | queue step `uid` |
| `pstart` | `chart_financial_reporting__services` (7404) | `prescription_start_time` | service `_id` |
| `dx_event` | same | `check_in_time` | service `_id` |
| `pulse_note` | `individuals-prescriptions` | `uploaded_at`, `is_draft = false` | prescription uid |

Idempotency is a **partial unique index** on `(source_ref, type, at) WHERE source = 'warehouse'`,
added by `0047`. A warehouse event has no `bench_session`, so slice 2's replay key does not apply to
it — without this a re-run would have silently doubled every event. Proven on production: a second
load reported 0 written, 43 already existed.

`cue` gained one column, `source_ref`. That is the whole schema change.

**We nearly shipped the wrong name.** The consult-start type was briefly built as `consult_start`
before we caught it. It is `pstart`, as you wrote it. Worth knowing that `cue.type` has no CHECK by
design, so a name mismatch does not error — your fuse would have found nothing and said nothing.

## Three things you need to know, none of them good

**1. The warehouse has no room dimension. At all.**

`queue_token_steps.routing` carries `DOCTOR_UID` only. `assigned_bay` is null on every row that day.
`STATION_UID` is used for diagnostic stations, not consult rooms. "OPD 7" and "Cardiology OPD" are
not representable in the warehouse.

The tightest defensible filter was hospital, and **22 different doctors ran consultations at EHRC on
19 August**. So attribution is by `doctor_uid` and nothing else. We resolved both clinicians' uids and
filtered to them; the mapping corroborates itself, because `individuals-prescriptions` independently
renders one as Cardiology Non Interventional and the other as Internal Medicine Specialist.

This bears directly on your §10.1 — "every consult-mark window that contains a warehouse `pstart` or
`called_at` produces a visit with that `individual_uid`." Without the doctor filter, a seven-hour OPD 7
window contains 69 `pqm_called` events from 22 doctors. **The fuse must filter by doctor, and the
room-to-doctor mapping has to come from outside the warehouse.** Today it comes from the tape labels.

**2. "Pulse note" is not a name the product uses.**

We checked the Pulse monorepo rather than guessing from the schema. The phrase appears nowhere. The
object is a **prescription** — `individuals/{individual_uid}/prescriptions/{uid}` — and the warehouse
table is generated straight from that Firestore proto. It is a full consultation note, not a drug
order: presenting complaints, assessments, plan of management, free text.

`uploaded_at` is written in exactly one place, at submit, so `uploaded_at IS NOT NULL` is a reliable
"finalised" predicate. Four caveats for slice 4: an **addendum overwrites the row and rewrites
`timestamp`**, pushing the original into `prescription_history` — so `at` is a snapshot, though
`source_ref` is stable, which is why idempotency survives it. Drafts are hard-deleted after two
months. **IPD notes are not in this table** — inpatient documentation is in the KareXpert
`kx_clinical_template_*` collections, so this is strictly the OPD note. And `consult_uid` can be
*inferred* at submit rather than authored, by picking the doctor's active calendar event from the last
day, so it is not always a real link.

We kept your cue type name `pulse_note`. Say if you would rather it matched the product.

**3. The gaps are the finding, and they are large.**

**35 of the 43 events fall outside every tape window.** OPD 7 has a **six-hour stretch** — after
04:36Z there is no warehouse activity for that doctor until 10:39Z — with live tape running
throughout. And Cardiology's 5 hours 7 minutes of tape correspond to **one** warehouse consultation:
one queue token, one note, one billed consult. V confirms that day was genuinely light, so the
warehouse is right and the room was mostly idle.

V's decision, which we have built to: **do not filter events by tape window.** The mismatch between
tape and warehouse is a first-class signal, not an error to be tidied away. Every cue carries
`in_tape_window` as an explicit boolean — `false` is written as a fact, never dropped.

## The payload slice 4 will read

```json
{ "source": "warehouse", "individual_uid": "…", "source_ref": "…",
  "attribution": "direct" | "inferred", "in_tape_window": true | false,
  "doctor_uid": "…", "category": "…", "at_source": "…" }
```

The last three appear only where the warehouse has them: `doctor_uid` on `pqm_called` only, `category`
on `pstart` and `dx_event`, `at_source` on `dx_event` and `pulse_note`. **An absent key is omitted,
never written as null**, so a null in a payload always means the warehouse gave us a null.

`attribution` is `direct` for 38 of 43. The 5 `dx_event` rows are `inferred`: non-consultation service
rows carry no doctor field of any kind, so they were attributed by the patient appearing in that
doctor's queue or prescription rows the same day. The two rooms' patient sets did not overlap.

## Two corrections to our own measurement, which you should have

Our §6.3 warehouse measurement of 17 August was wrong in two places, and we are revising it:

- **`HEALTH_PACKAGE` is a sixth diagnostic category** and was missing from the list. It is real — 25
  rows network-wide that day.
- **The `dx_event` COALESCE is dead.** `check_in_time` is non-null on 100% of qualifying rows, so
  `sample_collection_time` and `scan_in_time` never win. In practice `dx_event` is `check_in_time`,
  and it marks ordering or check-in, not the procedure — one LAB row had its sample collected 98
  minutes after the derived time.

## Two cliffs ahead, recorded now

- **`scribe_day_report` reads tape only.** No cues, no visits — it queries `bench_session`,
  `bench_chunk` and `bench_event` and never touches `cue`. So the §11.5 artefact you want, marks
  against warehouse against visits against tape, **does not exist yet**. It is a build, not a report
  we can run.
- **`visit.room_day_id` is single-valued and `NOT NULL`**, with no unique constraint on `visit` at
  all. Your §5 `visits[].room_day_ids`, plural, is **not representable** in migration 0042. A
  cross-room visit needs a schema change — a join table, or a thread above `visit`. Not slice 4's
  problem; slice 6's.

## Owed back from you

Slice 4 is the fuse, and you said you review this before it writes visits. Three questions:

1. Does the doctor-only attribution change anything in your §9 or §10?
2. Do you want `pulse_note` renamed to match the product, or left as specified?
3. §11.5's day report is a build. Do you want it before slice 4 or after?
