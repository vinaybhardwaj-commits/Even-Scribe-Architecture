# Desk consent + patient-voice privacy block
## Product Requirements Document

| | |
|---|---|
| Status | Direction **approved**. Hard constraints first-class. Builder-facing. |
| Date | 15 September 2026 |
| Author | Scribe Designer (from Vinay + Designer lock) |
| Audience | Builder **orchestrator**. Cut coder briefs from this. Not a ticket list. |
| Product repo | `Even-Transcription-Assistant`. Designer never pushes there. Designer does not implement. |
| This repo | `Even-Scribe-Architecture`. Design only. |
| Why now | Staff asked to blacklist certain people’s voices at the OPD front desk for privacy, without tracking consult start/stop. Voice is the gate. |

Cut briefs yourself. Do not treat this file as a sprint board. Consent **copy** is counsel’s. This PRD specifies mechanics only — what Scribe does when staff run the desk script and when a refused voice appears in an OPD room.

---

## 1. Summary / problem

OPD rooms already run continuous day tape. Patients who refuse recording still walk into those rooms. Today there is no product path that (a) hears the refusal at the desk, (b) remembers **that voice**, and (c) hides **that speaker’s** contribution from STT, emotion, and encounter export — without blanking the whole room as silence.

**Job:** Enroll desk staff, hear the patient’s short reply at a TONOR on the front desk, mint a centroid from that reply, and if they refuse, register a same-day hospital-wide blacklist. When that voice later scores in any already-recording OPD room, mark those turns `privacy_blocked` and keep doctor/other speech.

Voice identity is the privacy gate. Do not require staff to mark consult start/stop for this path.

---

## 2. Goals / non-goals

### Goals (v0)

1. Add a **desk room** with a TONOR. Same tape contract as other rooms: continuous IST day, five-minute pieces, native + bench, `room_day` opens with tape.
2. Enroll **staff** voice centroids **source-conditioned** (same pattern as clinicians: `enrolled_from` / `match_source` / mic). Desk TONOR is a source, not “the” centroid.
3. Natural desk flow: staff speak a short script (we’re recording; asking permission) → patient is heard acknowledging or refusing → **always mint** a centroid on the new (non-staff) voice from that reply.
4. Refuse → blacklist that patient centroid. Acknowledge → allow / soft-allow for STT and brain.
5. Propagate the block so **already-recording OPD rooms** can identify the voice and hide **that speaker’s** turns.
6. Fail **closed** when the desk said refuse and OPD match is unsure (D1).
7. Desk enroll / mint / blacklist hot path runs **local Mini or LAN**, not the public Cloudflare tunnel (D5).
8. Deferred delete of blocked audio spans; sealed audit of when/who blocked, without keeping audio forever (D6).

### Non-goals (v0) — D7 plus standing locks

- Cross-day **permanent** blacklist from one short desk utterance. Same-day IST first; durable only after match quality is proven (D4).
- Whole-room silence substitution (D2, D7).
- One desk embedding that magically matches every OPD mic (TONOR vs C270 vs others) without a fail-closed unsure path (D1, D7).
- **OT** (`room_7egw2my6` OT 3 and any future OT). Out of scope.
- Pulse writes. Slack writes. Invented legal/consent policy. Designer-written consent copy.
- Replacing Room Recorder, brain fuse, or the operator STT door.
- Live 250 ms clinic ear as a new product. Desk hot path is local VAD + staff match + embed; OPD hide is matcher + turn gates on the existing diarize/STT pipeline.
- Free-form “please don’t record me” as the blacklist command (D3).
- Tracking exact consult start/stop as a privacy prerequisite.
- Reopening clinician phone-print enrollment as the desk source. Desk staff enroll at the desk mic.

---

## 3. Actors & surfaces

| Actor / surface | Job in this PRD |
|---|---|
| **Desk TONOR room** | New Bench room. Continuous day tape like OPD. Placeholder: `room_desk_…`, session `bs_…`. Mic class `tonor` (or whatever the device reports). Not an OPD consult room. |
| **Desk staff** | Enrolled speakers. Speak the script. Issue the blacklist command (`staff centroid` + fixed phrase) or use the **desk-only** control. Not clinicians unless the same person is enrolled as staff for this source. |
| **Patient** | Unenrolled voice at the desk. Short yes/no (or equivalent acknowledge/refuse) is heard. Centroid minted from **that reply**, not from the staff script. May later appear in an OPD room. |
| **Attender** | Failure mode. May speak instead of or with the patient. Do not silently bind the wrong voice. See §10. |
| **OPD rooms** | Existing recording rooms (OPD 1/3/7, Cardiology, etc.). Match against blacklist using **that room’s source**. Apply `privacy_blocked` per turn. Doctor/other speech may remain. |
| **Operator** | Sealed audit read; deferred-delete status; desk Mini health; fail-closed counts. No requirement to sit the desk. Operator may have a **desk-only** control as D3 alternate, not a click-heavy consent UI. |
| **Home Mini / Desk Mini** | Local inference host. LAN preferred. Public tunnel is **not** the desk hot path. |
| **Builder** | Server + native. Designer never implements. |

Ignore OT unless it appears in a “do not wire this” test.

---

## 4. Consent state machine

One desk episode is a **consent take**. Staff do not click a wizard. Scribe watches the desk tape.

```
idle
  → staff_script_heard     (staff centroid matches; script window open)
  → patient_reply_heard    (non-staff speech in the reply window)
  → centroid_minted        (always, from the reply)   // D4
       ├─ reply = refuse   → blacklist_armed          // D4
       │                      + command or desk control confirms if required by D3
       └─ reply = acknowledge → allow / soft-allow    // D4
  → published_to_opd_matchers
```

### 4.1 Script

Staff speak a **short, fixed script** to the patient: recording is on; asking permission. Exact words are counsel’s. Product needs:

- A **detectable** staff-script window (staff centroid + expected duration band).
- A **reply window** immediately after: first non-staff voiced span.

Do not STT the patient’s reply on the **public** path to decide yes/no if that would send desk audio out the tunnel. Classify refuse vs acknowledge on the **local** desk path (keyword/intent on-device, or staff command — see 4.4). Builder chooses the local classifier; fail closed if the reply is unclassifiable **and** staff have not issued an explicit allow.

### 4.2 Hear reply

Hot path (D5):

1. **VAD** on the desk TONOR.
2. **Staff vs other** using enrolled **desk-source** staff centroids.
3. **Embed** the non-staff reply → mint `cen_…` (patient desk print, source = desk TONOR).

Full **pyannote** turn-split on the desk is **optional**, only if the desk is chaotic (overlapping queue). Default is VAD + staff match + one embed. Do not send this hot path through `DIARIZE_BASE_URL` over the public tunnel.

### 4.3 Always mint

Every classified reply mints a centroid, refuse or acknowledge (D4). Mint is not the blacklist. Mint is the identity handle.

- Refuse without a usable embedding: **do not** invent a centroid. Fail closed for that take: treat as **unmatched refuse** — OPD rooms that cannot score this voice still must not export **unsure** patient-like speech if the operator later attaches identity another way. For v0, log `mint_failed` and require a retry at the desk or a desk-only control. Do not guess.
- Acknowledge with a usable embedding: `cen_…` status `allow` or `soft_allow` for that IST day (builder: `soft_allow` = may STT/brain; not a durable enroll into clinician tables).

Patient centroids **do not** go into the clinician voiceprint table. Separate registry (§5).

### 4.4 Blacklist command (D3)

Blacklist is **not** free-form chatter and **not** “anyone said privacy.”

v0 trigger (either or both; both must be **high precision**):

1. **Staff centroid** (desk source) + **fixed phrase** (example: “Scribe, privacy block”). Phrase is locked in code. Not a fuzzy LLM parse of the consult. Nearby patient speech must not fire it.
2. **Desk-only control** — one control on the desk surface or operator desk pane. Not on the OPD consult lamp. Not a Pulse button.

Refuse classification from the patient’s reply **plus** the staff command is the intended pair: hear refuse → staff confirm with the phrase (or the control) → registry write. If Vinay wants refuse-alone (no phrase) after the local classifier is proven, that is a later tightening — **v0 ships the command** so false triggers stay rare.

Acknowledge does **not** require a command. Mint + allow/soft-allow.

### 4.5 Timing vs OPD

The patient may already be in an OPD room, or walk in seconds later. Tape in OPD is already running. Privacy does **not** wait on consult marks.

Race: mint/blacklist must land on OPD matchers **fast** (LAN/local bus or brain registry the rooms already poll). Until it lands, see §10 race.

---

## 5. Blacklist registry

### 5.1 What it stores (no PHI in examples)

A **patient desk print** is not a Pulse row. No MRN, no name in this registry.

```
privacy_print
  id:                cen_…
  kind:              patient_desk
  status:            allow | soft_allow | blocked_same_day | blocked_durable | mint_failed
  enrolled_from:     room_desk_… | source_id
  match_source:      tonor | …          // enrollment source, not a universal key
  ist_day:           2026-09-15         // Asia/Kolkata
  minted_at:         …
  embedding:         (not in MCP chat, not in this repo)
  staff_actor_cen:   cen_staff_…        // who ran the take
  desk_session_id:   bs_…
  desk_span:         { start_ms, end_ms }  // reply used to mint
```

```
privacy_block
  id:                blk_…
  print_id:          cen_…
  scope:             hospital_ist_day | durable
  ist_day:           2026-09-15
  reason:            desk_refuse
  command:           phrase | desk_control
  armed_at:          …
  actor_staff_cen:   cen_staff_…
```

Clinician/staff prints stay as they are: source-conditioned (`enrolled_from` / `match_source` / mic). Desk staff get **desk TONOR** samples. Do not reuse a phone centroid as the desk staff print (same lesson as far-field doctor vs phone).

### 5.2 Same-day vs durable (D4)

| Scope | When | Where it applies |
|---|---|---|
| `hospital_ist_day` | **v0 default** on refuse | All **in-scope OPD** rooms that calendar-day (IST), until day end. |
| `durable` | **Not v0 default.** Only after match quality is proven across sources. | Explicit promotion. Not one short utterance (D7). |

Hospital-wide for v0 means **OPD recording rooms that day**, not OT, not Home Office as a clinic, not scratch fuse rooms.

### 5.3 Propagation to OPD matchers

Each OPD matcher scores **its own source** (that room’s mic) against the print.

**D1:** Desk centroid ≠ automatic room match. Same voice scores differently on TONOR vs C270 vs others.

Product bar: **multi-source coverage**.

v0 implementation rule:

1. Keep the **desk** embedding as the mint.
2. For each OPD **source class** you claim to cover, you need either:
   - a **source adapter** / calibration so desk embeddings are not compared raw to a foreign mic, **or**
   - an **unsure** band that **fail-closes** (treat as blocked for export) when `privacy_block` is armed and score is not a clear miss.
3. Do **not** ship “one desk vector, cosine vs every room, hope.”

Clear **miss** (score below a miss floor): not this speaker; do not hide others.  
**Hit** (above hit floor for that source): `privacy_blocked` on those turns.  
**Unsure** (between floors) **and** a same-day/durable block exists: **fail closed** — hide the candidate patient-like turn (or the unmatched non-staff turn in that window), do **not** STT/export it. Do not hide the matched clinician.

Threshold numbers are **calibration**, not this PRD. `SPEAKER_MATCH_THRESHOLD` for clinicians stays a separate constant. Do not reuse 0.78 as a binder here.

### 5.4 Allow / soft-allow

Acknowledge → print `allow` or `soft_allow`. OPD STT and brain **may** hear that voice that IST day. This is not a clinician enroll. It is not a permanent patient voice bank.

---

## 6. OPD runtime behavior

### 6.1 Hide semantics (D2) — locked

**Do not** blank the whole room as silence. **Do not** write `stt_silence` for the consult because one speaker refused.

Redact **that speaker’s turns**:

| Gate | `privacy_blocked` turn | Other speakers |
|---|---|---|
| STT | Skip. No transcript text. | May run as today. |
| Emotion / sentiment | Skip. | May run as today. |
| Encounter export / note / fuse consumer that quotes speech | Skip this speaker. | Doctor/other may remain. |
| Tape | Still recorded until deferred delete (§9). Hide is an **output** gate first. | Unchanged. |

Cue/turn shape (illustrative — builder maps onto existing `stt_turn` / `speaker_match` / new type):

```
type: privacy_blocked
at: …
payload: {
  end,
  print_id: "cen_…",
  block_id: "blk_…",
  speaker_idx,
  reason: "desk_refuse",
  match: "hit" | "unsure_fail_closed",
  match_source: "c270" | "tonor" | …,
  text: null
}
```

Do not put patient words in `payload`. Do not paste them into Slack, MCP chat, or this repo.

### 6.2 Downstream paraphrase (D2)

Doctor/other speech **may remain**. Downstream **must not** reintroduce the blocked patient’s content via **doctor paraphrase** into exports.

v0 rule for encounter export / note / any packaged output:

- If a window contains `privacy_blocked` turns, **export must not include** clinician restatement of what the patient said in those spans (symptoms quoted, history repeated, “patient says…” tied to the blocked interval).
- Builder: either drop clinician turns that overlap/follow a blocked patient span under a **same-utterance / immediate paraphrase** heuristic, **or** hold those windows out of export until a later slice. **Fail closed** on export for that visit window if you cannot separate paraphrase from original clinician speech.
- Brain fuse: `privacy_blocked` is **not** visit-mint evidence and **not** STT evidence. Do not promote blocked text through Arm A or any arm.

This is a product lock, not a “best effort filter.”

### 6.3 Continuous tape, no consult clock

OPD rooms keep recording. Privacy block is **speaker + time span**, not pause-the-room. Staff do not need to Pause for this path (Pause remains a separate consent verb on the lamp; copy is counsel’s; this PRD does not replace it).

### 6.4 Who-in-room

Pyannote turn-split + embed/centroid for who-in-room stays the OPD path (Mini diarize today via tunnel for **async OPD replay** — D5). Desk hot path does not use that tunnel.

When clustering is not running (`clustering_not_running` as of 15 Sep 2026 on live OPD 7), **this feature still requires** a per-turn speaker embedding on OPD tape for hide to work. Builder owns turning on `diarize_window` / speaker match for **live or near-live** OPD enough to gate STT — or gating STT until a turn is classified. Fail closed: **unclassified non-staff** speech in a room while a hospital-day block list is **non-empty** is **not** exported until classified or scored as a miss. Empty block list → today’s behaviour.

---

## 7. Desk ML placement (D5)

| Path | Where it runs | What |
|---|---|---|
| Desk hot path | **Local Desk Mini** or **LAN to home Mini** | VAD → staff vs other (desk staff centroids) → embed non-staff reply → allow/block registry write |
| Desk chaotic | Same local/LAN | Full pyannote **optional** |
| Async OPD replay / diarize_window | Existing Mini via tunnel **allowed** | Who-in-room for OPD tape, matcher vs registry |
| Public Cloudflare tunnel | **Forbidden** for desk enroll, mint, blacklist command, reply embed | Do not put desk consent audio on the public hear path |

`DIARIZE_BASE_URL` / pyannote health on the map (`pyannote-3.1`, `ecapa-voxceleb`) is the **OPD/async** stack unless the same process is reachable **on LAN** for the desk. If the only route is the public tunnel, **desk v0 does not ship** that call. Build LAN or colocate a Mini at the desk.

Do not add a second public ffmpeg/STT stack for the desk.

Staff enrollment: same local path. Source-conditioned samples, like clinicians (`enrolled_from`, `match_source`, mic). Example of existing clinician enroll shape: curated TONOR session ids already exist for doctors; desk **staff** are a distinct enroll set even if a person is also a clinician.

---

## 8. Data model sketches

Placeholder ids only. No names, no MRN, no transcript.

```
room_desk_…                 desk TONOR room
bs_…                        desk or OPD session (day tape)
rd_…                        room_day
cen_staff_…                 staff desk-source centroid
cen_…                       patient desk print (mint from reply)
blk_…                       privacy_block row
doc_…                       clinician (unchanged table; not the patient registry)
vs_…                        voice sample (staff enroll only here)
```

**Do not** store patient Pulse ids on `privacy_print`. Linkage to a visit, if ever needed, is a **later** product lock and still not a Pulse write from this path.

Illustrative take:

```
consent_take:
  id: take_…
  desk_session: bs_…
  staff_cen: cen_staff_…
  reply_span: { start_ms: 12_400, end_ms: 14_900 }
  print_id: cen_…
  outcome: blocked_same_day | allow | mint_failed
  block_id: blk_… | null
```

OPD match event (audit + matcher, not STT text):

```
privacy_match:
  id: pmatch_…
  block_id: blk_…
  room_id: room_…
  session_id: bs_…
  span: { start_ms, end_ms }
  match: hit | unsure_fail_closed | miss
  match_source: c270
```

---

## 9. Audit & deferred delete (D6)

### Audit (sealed)

Keep **when / who blocked / which print / which rooms matched** without keeping the audio forever.

- Actor = staff centroid + desk control id if used. Not a Slack ping.
- No audio bytes in `audit_log`. No transcript of the patient.
- Operator MCP may list counts and ids (`blk_…`, `cen_…`, room, IST day, `hit|unsure_fail_closed`). Default summaries. No `include_payload` of embeddings.

Existing `scribe_audit_recent` is install/MCP-shaped. Do not overload it with patient speech. New audit actions (names illustrative): `privacy.block_arm`, `privacy.match`, `privacy.delete_span`.

### Deferred delete

1. **Output gate first** (`privacy_blocked`) so live STT/export stop even while bytes still sit on R2.
2. **Then** delete **blocked spans** from cloud (desk reply used to mint **and** OPD spans that hit / fail-closed). Piece-level delete must not punch a hole that looks like “the room was silent.” Remaining audio in the piece stays. If the object is a five-minute piece, **rewrite or split** so other speakers remain; do not replace the whole piece with digital silence (D2).
3. After delete, audit retains the span clocks and block ids, not the waveform.
4. Deletion is **deferred** (batch after the take, after the room-day, or a job). v0 may delay to end of IST day. It must still be **scheduled and observable**. “We will delete someday” is not done.

Unblocked speech in the same piece is **not** deleted.

---

## 10. Failure modes

| Failure | v0 behaviour |
|---|---|
| **Cross-mic miss** (desk TONOR print vs OPD C270) | Expected (D1). Unsure + block armed → fail closed (hide candidate turn). Clear miss → do not hide. Do not claim universal match. |
| **Attender voice** | Reply embed may be the attender. Blacklist then hides the **wrong** person and misses the patient. Mitigation: staff script + command only when the **patient** just spoke; optional second mint if two non-staff voices in the reply window — if two, **fail closed** (do not pick); staff retry or desk control on a pointed span. Do not auto-pick the louder speaker. |
| **Race before mint lands** | OPD may STT the patient for seconds. Buffer/hold **non-staff** OPD turns for a short publish window **or** retrospectively strip STT/export once `blk_…` lands (delete text from cues; leave `privacy_blocked`). Live encounter export must not ship those turns. Prefer retrospective strip + hold-off export over dropping the room. |
| **False blacklist command** | Fixed phrase + staff centroid (D3). Patient or TV must not fire it. Desk-only control is the escape hatch. Log and make staff undo: `blocked_same_day` → `allow` for that print that day (operator or desk control). Undo is required in v0. |
| **Mint failed** (too short / noise) | No centroid. No silent succeed. Retry at desk. |
| **Empty block list** | Do not fail-closed unclassified speech hospital-wide. Today’s OPD behaviour. |
| **Staff not enrolled** | Desk path cannot separate script from reply. Refuse to arm a block from command-only without a patient embed. Enroll staff first. |
| **Tunnel-only Mini** | Desk hot path **does not ship**. LAN/local Mini required (D5). |
| **Paraphrase leak** | Export hold or drop overlapping clinician restatement (D2). Fail closed on that window’s export. |
| **OT** | Not in matcher scope. |

---

## 11. Acceptance criteria / test plan

Designer does not run clinic. Builder proves on Home Office / scratch / a quiet desk fixture **without** writing live OPD `stt_turn` until Vinay says the live day is safe. Prefer scratch room-days for STT writes, consistent with the operator door.

Do not paste transcripts into the report. Send ids, spans, match bands, gate results.

| # | Pass |
|---|---|
| A1 | Desk room records a day tape: pieces verify, `room_day` exists with zero consult marks. |
| A2 | Staff enroll on **desk TONOR** is source-conditioned. Phone/clinician print is **not** used as the desk staff key. |
| A3 | Script + non-staff reply **always mints** `cen_…` on acknowledge and on refuse when audio is usable. |
| A4 | Refuse + D3 command (phrase and/or desk control) writes `blk_…` scope `hospital_ist_day`. Acknowledge does not. |
| A5 | Command without staff centroid does **not** arm. Phrase from a non-staff speaker does **not** arm. |
| A6 | OPD room, **same source class as desk**, hit → those turns are `privacy_blocked`: no STT text, no emotion, not in encounter export. Clinician turns in the same window **remain** unless paraphrase rule holds them. |
| A7 | OPD room, **different mic class**, calibrated unsure + block armed → fail closed (hidden), not exported. Clear miss → not hidden. |
| A8 | Whole-room `stt_silence` / digital silence substitution **does not** occur because of a block. |
| A9 | Desk hot path does not call the **public** tunnel (prove with logs / base URL). LAN or local Mini only. |
| A10 | Pyannote on desk is off in the quiet-desk fixture; optional path documented, not required for A3. |
| A11 | Undo block same IST day restores allow for that `cen_…`. |
| A12 | Two non-staff voices in the reply window: no auto-pick; mint_failed or explicit retry. |
| A13 | Race: STT text that landed before `blk_…` is stripped from export; cues become `privacy_blocked`. |
| A14 | Deferred delete job removes **blocked spans** from cloud; audit row remains without audio; neighbouring speech in the piece still exists. |
| A15 | No Pulse write. No Slack write. No OT matcher. No durable promotion in v0. |
| A16 | Empty hospital-day block list: unclassified OPD speech is not mass-redacted. |
| A17 | Operator can read sealed audit (ids, times, actor staff cen, match hit/unsure) without embeddings or patient text. |

Home Office may be used as a **mic lab**, not as a visit mint. Do not fuse Home Office.

---

## 12. Open questions for Vinay / builder

1. **Desk Mini:** New Mac at the desk, or LAN to the existing home Mini? v0 cannot use public tunnel either way.
2. **Refuse-alone vs command:** v0 requires D3 command. When (if ever) is local yes/no enough to arm without the phrase?
3. **Paraphrase:** Hold the whole overlapping clinician window out of export, or ship a heuristic? Fail-closed hold is the default if unset.
4. **Publish bus:** How do live OPD tabs/native recorders learn `blk_…` within seconds — existing command poll, brain cue, LAN fan-out?
5. **Delete SLA:** End of IST day vs N minutes after last match? Piece rewrite vs span-scrub in object storage?
6. **Soft-allow retention:** Are acknowledge prints dropped at IST midnight with blocks, or kept only in sealed audit?
7. **Staff set:** Front-desk roster vs any enrolled clinician who stands at the desk that day?
8. **Matcher calibration:** Who records the TONOR↔C270 (and other) pairs, and who sets hit/unsure/miss floors? Designer will not invent thresholds.
9. **Live vs scratch:** First clinic morning on live OPD hide, or scratch-only until A6–A8 pass on a fixture?
10. **Undo actor:** Desk staff only, or operator too?

---

## 13. Decisions table (D1–D7)

| ID | Decision | v0 rule |
|---|---|---|
| **D1** | **Cross-mic** | Desk centroid ≠ automatic room match. Product bar is **multi-source coverage**. If desk-refused and match **unsure**, **fail closed**. |
| **D2** | **Hide semantics** | Do **not** blank the whole room as silence. Redact that speaker’s turns (`privacy_blocked`): skip STT / emotion / encounter export for those turns. Doctor/other speech may remain. Downstream must **not** reintroduce patient content via doctor paraphrase into exports. |
| **D3** | **Blacklist command** | Not free-form chatter. **Staff centroid + fixed phrase** (e.g. “Scribe, privacy block”) and/or **one desk-only control**. Avoid false triggers. Undo required. |
| **D4** | **v0 policy** | Always mint from the yes/no reply. Refuse → blacklist (**hospital-wide that IST day** first; **durable** only after match quality proven). Acknowledge → allow / soft-allow for STT/brain. |
| **D5** | **Local desk inference** | Desk enroll / blacklist must **not** use the public Cloudflare tunnel. Local Desk Mini or LAN to home Mini. Hot path: VAD → staff vs other via staff centroids → embed non-staff reply → allow/block. Full pyannote only if desk is chaotic. Tunnel reserved for **async OPD replay**. |
| **D6** | **Deletion** | **Deferred delete** of blocked spans. **Sealed audit** (when / who blocked) without keeping audio forever. |
| **D7** | **Out of v0** | Cross-day permanent blacklist from one short utterance. Whole-room silence substitution. Single desk embed catching every OPD mic without a fail-closed unsure path. |

Direction approved. These constraints are first-class. Do not “simplify” D1–D2 in a first slice.

---

## Locks that stay closed

- Designer does not implement. Builder owns server and native.
- No Pulse writes from this privacy path. Not a future footnote — **out** until a later PRD says otherwise.
- No Slack writes. No raw R2 keys in chat or this repo. No patient transcripts in this repo.
- Consent copy is counsel’s.
- Encounter unit remains: **patient OPD visit that IST day**. Tape remains **per room**. Brain fuses cues. This path does not mint visits from desk audio.
- Clinician centroids remain source-conditioned. Do not collapse sources into one vector.
- Operator STT/brain door PRD (22 Aug 2026) is not replaced. Hear tools still must not dump clinical speech into chat by default.
- OT out of scope.

---

## What comes back to the designer

- Commit + sha (preview or production).
- Which acceptance rows pass; which are blocked on Mini placement (Q1).
- Desk path base URL evidence (LAN vs tunnel).
- Cross-mic: which source classes were tested (TONOR, C270, other), and whether unsure fail-closed fired.
- Whether paraphrase export was hold-all or heuristic.
- No transcripts. No embeddings. Ids and counts only.
