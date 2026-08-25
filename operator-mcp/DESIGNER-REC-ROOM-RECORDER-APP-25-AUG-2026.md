# Designer recommendation — Room Recorder app
25 August 2026. Scribe Designer. For the orchestrator and the coder.

This is the recommendation. It **supersedes** `DESIGNER-REPLY-KIOSK-APP-AND-25-AUG-STATE-25-AUG-2026.md` and `DESIGNER-AMEND-NATIVE-RECORDER-25-AUG-2026.md`. Those two files argued a wrapper. That was wrong. Do not build a wrapper.

This is not a coder ticket list. Cut briefs from the locks below.

---

## Build this

An independently installed **Room Recorder** on the clinic Mac.

Not a browser tab. Not the current room page inside Electron, Tauri, or WKWebView. Those are still browsers. Every operational failure on 24 August was a browser failure.

The app:

- Installs on our hospital Macs.
- Launches at login, before anyone arrives.
- Restarts itself if it dies.
- Comes back after a power cut.
- Cannot be closed by a passer-by.
- Holds the machine awake while recording.
- Keeps reporting to the operator whether or not it is recording.
- Can be restarted remotely as a process.

Capture is native. The process owns the microphone.

---

## Do not rebuild the tape

The server contract does not change.

Five-minute pieces. Same content-type. Same upload, retry, verify. Same R2 keys. Same command bus. Same brain. Same operator page (except as noted below).

If native capture cannot emit a piece the server already accepts, the capture is wrong. Do not invent a new audio format, a new rotation, or a new stitch.

Hardening the Macs (no sleep, auto-login) is required for the app to stay up. It is not a substitute for the app.

---

## Room screen

A lamp, not a dashboard.

**Consult face.** State: off / recording / paused / finished for today. Three verbs: Start, Pause, Mark consult. Nothing else.

- Start is Start Recording, not Pulse start-prescription.
- Pause is consent. Copy is counsel's. Designer does not write it.
- Mark posts `consult_mark` with room + timestamp only. No patient ID.

**Not on that screen.** Piece counts, timers as the hero, R2, gaps, doctor clock, warehouse silence, stranded minutes, operator alarms, standing level meters, transcript queues.

**Setup overlay.** First launch, or a hidden gesture (long-press the lamp). Mic name, a level meter, and whether a spare device is **present**. For the person plugging in at 8am. Not the consult face.

A black screen with a pip is allowed if counsel wants less theatre. The three verbs must still be reachable without a browser. Do not hide Start or Pause.

---

## Day, marks, remote control

The `room_day` opens when the tape starts (first piece or `start_day` ack). A recording with no day record is a bug. Ship this **before** the next clinic morning. It does not wait on the app.

Marks are consult boundaries. They do not make tape processable. Tape on R2 is enough. Processing must not wait on a tap. 19 August holes had no marks and were still walked.

Remote Mark consult already exists. It is **fallback**. The operator uses it when they can hear the room and they say why. It is not the plan and not how audio becomes processable. Do not add a second primary Mark on the operator page.

Remote restart of the **app** is in. Remote start/stop/pause of the **tape** stays listener-gated, consent-aware, and not used on a room with patients unless Vinay said so.

Verbal briefing remains the plan for doctors marking.

---

## Locks the app does not reopen

- Alarm rule: if a new alarm fires on a healthy room doing a normal thing, the alarm is wrong. Test against an ordinary day before it ships.
- Spare-exists means a device is present, not that a piece arrived. One microphone is normal. No empty lane. No amber vital for a webcam that is not there. Do not bind a window to the spare on a flag alone.
- Doctor clock stays hidden where nothing feeds it. Warehouse has no room dimension. Monday join is (c).
- Finished-for-today is not a kiosk-dropped fault.
- No Pulse writes. No Slack writes. No live `stt_turn`. No visit mint from Home Office or from this app.
- Voice / diarization / enroll is later. Do not sit doctors down. Do not use phone prints.
- 22 Aug STT/brain door PRD stands. Hear 24 August with Whisper (Cardiology `bs_z3gpbh6e`, OPD 5 `bs_8294hb7f`). Paid engines stay on-ask.

---

## Acceptance

| # | Pass |
|---|---|
| 1 | Installed Mac app. Not a tab. Not a wrapped page. |
| 2 | Launch at login, self-restart, return after power, not closable by a passer-by. |
| 3 | Native capture produces pieces the **current** server verifies. Same content-type. No new format. |
| 4 | Consult face is lamp + Start / Pause / Mark. No telemetry. |
| 5 | Setup overlay shows mic name, meter, spare-present. |
| 6 | Reporting continues when not recording. Operator can restart the process. |
| 7 | `room_day` exists once tape starts, with zero marks. A 55-minute take without a tap is processable. |
| 8 | Archive, API, brain, STT path untouched except the day-opens-with-tape fix. |

---

## What comes back

- What you built (app name, how it is signed and installed).
- One Home Office day through the app: pieces verified, day record present without a mark.
- Any place native capture could not match today's piece contract (name it).

Do not send a wrapper and call it a first slice.
