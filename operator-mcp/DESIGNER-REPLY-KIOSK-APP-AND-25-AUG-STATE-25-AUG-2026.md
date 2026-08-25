# Designer reply — kiosk as a Mac app, and the 25 Aug state
25 August 2026. Scribe Designer. For V and the orchestrator.

Read V's note of the same morning. Four questions. All closed. This is not a kickoff and not coder tickets.

---

## Ground I checked, not only the note

24 August on the door:

| Room | Session | IST (approx) | Tape |
|---|---|---|---|
| Cardiology OPD | `bs_z3gpbh6e` | 11:58–16:16 | 52/52 primary, 52 backup, 21 ms gap, `primary_lost` 1, `mic_status` `on_backup` |
| OPD 5 · Salanki | `bs_8294hb7f` | 11:55–16:16 | 53/53 primary, 40 backup, 20 ms gap, `primary_lost` 1, `mic_status` `on_backup` |
| OPD 5 · Salanki | `bs_f46u4jxw` | 11:30–11:35 | short take, also `on_backup` |

That is the 8 hours 44 minutes. Both clinic days ended on the spare. The recording layer is fine. I agree.

This morning (25 Aug ~09:10 IST): Home Office is finished for today, kiosk listening, 57 minutes on tape, one mark, transcript lane on with 3 waiting, 45 minutes called stranded as `never closed`. Clinic rooms are offline since yesterday. That is the browser problem the proposal is about.

I last signed remount, the fuse, speech turns, the 19 Aug hole, Monday join (c), and the 22 Aug STT/brain door PRD. The operator page rebuild, the four false alarms, the spare-binding fault, and the 24 Aug clinic day are new to me. I am answering those now. I am not re-opening K3–K5 or the 22 Aug STT PRD.

---

## 1. Wrapper. Not the rewrite. Not "harden the Macs and stop."

The recommendation is right. I would push back if you started the rewrite now.

The only part of Scribe that has never let us down is the tape: five-minute pieces, upload, verify, 19 ms across twelve rotations, 8 hours 44 minutes with no gaps. A native rewrite puts that path back on the table. Do not.

Hardening the machines (no sleep, auto-login, open the page) is necessary **and insufficient**. It leaves the tab closable and the room unreachable, which is 24 August at three rooms and a walk downstairs. Do that work as well — a wrapper still needs a Mac that does not sleep mid-consult — but do not call it the product.

**The wrapper is the first move.** Same page, same `MediaRecorder`, same chunking, same command bus, inside a window the OS owns. Launch at login. Restart if it dies. Come back after power. Not closable by a passer-by. Hold the machine awake **while recording**. Keep reporting when it is not. Remote restart of the **process** is allowed. Remote start of the **tape** stays what it is today (listener required, no invented session).

If the wrapper fails, the failure must be named (mic permission lost after reboot, update killed capture, WKWebView dropped the device). Then we talk about a rewrite. Not before.

Do not touch the archive, the API, the brain, or the STT path to do this. Audio format parity is free because it is the same code.

---

## 2. The room screen is a lamp, not a dashboard

You asked me, so this is the design. Do not ship today's admin-ish room page inside the app and call it done.

**Patients and doctors see three things.**

1. State: off / recording / paused / finished for today. A lamp. Not a timer as the hero. Not a piece count.
2. The three verbs the human still owns: **Start**, **Pause** (consent — copy is counsel's, I will not write it), **Mark consult**. Same verbs as now. Distinct Start, not Pulse start-prescription. Mark still posts `consult_mark` with room + timestamp only. No patient ID on the kiosk.
3. Nothing else.

**They do not see:** piece counts, R2, gaps, doctor clock, warehouse silence, stranded minutes, operator alarms, level meters as a standing widget, "3 waiting to transcribe."

**Setup overlay, not the consult face.** First launch, or a hidden gesture (long-press the lamp). Mic name, a level meter so the person plugging in can see it hear, and whether a spare device is **present**. This is how a tech sets the room at 8am. It is not what sits on the desk during a consult.

A black screen with only a pip is allowed if counsel would rather the room not perform "you are being recorded." The three verbs still have to be reachable without opening a browser. If you hide Mark too well, we are back to one mark in seven hours. Do not hide Start or Pause.

I am not restyling chrome for its own sake. I am taking telemetry off a screen that faces patients.

---

## 3. Remote mark is fallback. Making tape processable is not a mark.

Two different problems got glued together in the note.

**A. The day must exist when the tape starts.** A recording that creates no day record is a product bug. Marks are consult boundaries, not a permission slip for the day to exist. First piece (or `start_day` ack) opens the `room_day`. If 55 minutes sat stranded on 25 August until someone pressed a button in the room, that button was doing the day's job. Take that job off Mark.

Processing must not wait on a mark. We already walked 19 August holes that had no marks. Window-as-unit does not ask whether anyone tapped. If the current pipeline keys "can transcribe" on a cue, change that. Tape on R2 is enough.

**B. Remote Mark consult.** The tool already exists (`scribe_mark_consult`). It stays **fallback**. The operator uses it when they can hear the room and they say why (same fence as 22 Aug). It is not the plan, not the briefing, and not how audio becomes processable. Do not add a second mark button on the operator page that looks like the primary lever.

Verbal briefing of the doctors remains the plan for marks. One mark in seven hours on 19 August did not get better because we moved the button to the desk.

Remote **restart of the app** is a different verb and I want it. Remote **start/stop/pause of the tape** stays listener-gated, consent-aware, and not used on a room with patients unless Vinay said so.

---

## 4. What to stop, and what I will hold you to

**The alarm rule is now a lock.** Check every new alarm against an ordinary day before it ships. If it fires on a healthy room doing a normal thing (end of day, start of day, a quiet afternoon, a deliberate stop), the alarm is wrong. I will reject the next one that fails that test.

Stop these, specifically:

- Shipping an end-time alarm that fires on every ordinary end of day.
- Showing a doctor clock where nothing feeds it, or falling back to tape length and wearing a clock-gap label. Hide it. The warehouse still has no room dimension. Monday join is still (c).
- Treating "kiosk dropped" as a fault when the day was ended on purpose. That is *finished for today*. You already added the state. Keep it.
- Binding a window to the spare on a flag alone. You say this is fixed. Hold it. 24 August both clinic sessions still show `on_backup` / `primary_lost` 1. I want the next report to say whether those windows were rebound to primary or left on spare **because the size test and the meter agreed**, not because a flag stuck.
- Inferring that a spare exists because a piece arrived. A 70 KB "backup" from a phantom device is not a microphone. Spare-exists means a device is present. One microphone is normal, not degraded. No empty lane, no amber vital for a webcam that is not there.
- Sitting doctors down to read sentences for a voiceprint. Phone-at-arm's-length already failed here (0.55–0.62 against a 0.78 bar we banned). Enroll from room tape, later. Slice B is still later. Do not run diarization on the dormant phone-encounter path and call it the room.
- Waiting on a paid engine, by hand, before 24 August is heard. Whisper on the Mini is free and already walked 19 August. A person asking each pass is the right default for Deepgram / Sarvam / ElevenLabs. It is the wrong default for Whisper on a finished clinic day. The 22 Aug STT door PRD (S0–S2) is how the operator orders that. 8 hours 44 minutes of unheard clinic tape is now the first Whisper walk, not a budget meeting.

I have not read the thirty-nine voice decisions. I am not ratifying a stack I have not seen. The four you listed (room-tape enroll, patient holds a set, roles from recurrence, two-visit confirm) already match locks from 18–22 August. Send the thirty-nine if you want them held.

Do not stop: truth on the operator page (finished, stranded in minutes, meters, size-vs-baseline). That work stays.

---

## What I am not asking you to build in this reply

- A native rewrite.
- New clinician chrome beyond the lamp and the three verbs.
- A scheduled paid-engine fan-out.
- Voice / diarization / enroll.
- Pulse `doctor_opd_rooms` or a 52-doctor dump.
- Consent copy.

The wrapper can start. The day-record-on-tape-start fix should ship **before** the next clinic morning, in the page or the app, because it is independent of the wrapper and it is how 55 minutes get stranded.

---

## Answers, one line each

1. Wrapper first. Rewrite only after the wrapper fails for a named reason.
2. Lamp + Start / Pause / Mark. Telemetry off the consult face. Setup overlay for the meter.
3. Yes the operator may mark, as fallback, with a reason. No: a mark is not what makes tape processable. The day opens when the tape starts.
4. Stop shipping alarms that fire on healthy days. Stop spare-from-a-flag and spare-from-a-piece. Stop treating one mic as broken. Hear 24 August with Whisper. Do not enroll from phones.
