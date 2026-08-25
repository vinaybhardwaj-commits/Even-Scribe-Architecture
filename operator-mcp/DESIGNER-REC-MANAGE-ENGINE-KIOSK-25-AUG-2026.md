# Designer recommendation — Keep the browser up with Manage Engine
25 August 2026. Scribe Designer. For the orchestrator, Kush, and IT (Satya / Pruthvi).

This **supersedes** `DESIGNER-REC-ROOM-RECORDER-APP-25-AUG-2026.md` as the next build. The native Room Recorder app is deferred. It is a nice-to-have if this fails. Do not start it.

The browser kiosk is already in the rooms. The failure from standup is that people close or minimize it, or the desk says the room is dead while the page is still up. Fix that with software Even already has on the Minis.

---

## Build this, not an app

A **watchdog** on each clinic Mac that notices the room page is gone and opens it again.

Manage Engine (Endpoint Central / Desktop Central) is how we **install and run** that watchdog. It is not the heartbeat itself.

Even already uses it. Console: `desktopcentral.manageengine.in`. Satya (`satyajeet.nag`) and Pruthvi (`pruthvi.rd`) own the agents. `#it-support` is the channel. There is no Manage Engine connector for this chat. We write the script. IT deploys it.

---

## Why not a 90-minute refresh

ME Mac custom scripts can run at startup, at logon, once, or on the agent **refresh cycle (~90 minutes)**. A clinic morning cannot wait 90 minutes for a dead tab.

So:

1. ME deploys **once**: a `launchd` plist + a small `sh` that checks every 30–60 seconds.
2. The local job is the heartbeat. ME is the truck.
3. Same script can be re-run from ME as a Mac Custom Script (Configurations → Add → Mac → Custom Script → Computer) if we need a one-shot "open the room now."

Do not use ME's refresh cycle as the watchdog.

System Manager remote command prompt is a **Windows** tool. Do not plan on it for the Minis.

---

## What the watchdog does

On each target Mini:

- Know the room URL (one slug per machine).
- Check that a browser is running **and** that our listener is fresh (the same signal `scribe_diff_room` uses: page open, not stale).
- If the browser is gone: open Safari or Chrome at that URL. Prefer the browser that already works in that room (standup: flipping to Safari brought a "dead" room back).
- If the browser is up and the listener is stale: reload the room URL. That is the "page is up, desk says dead" case. Do not treat it as healthy.
- Do not start the **tape**. Only the page. Start is still the human verb (or the existing remote start, listener-gated).
- Log what it did (reopened / reloaded / already ok) to a local file we can read later.

First machine: Home Office Mini. Then one clinic room. Not all rooms on day one.

---

## What this will not fix

- Consult start/end at 50%. That is the brain, not the tab.
- Teleconsult vs chair in Dibyendu's room. Same.
- A Mac off at the wall.
- Mic unplugged.
- A 90-minute ME refresh used as if it were a heartbeat.

If the watchdog is running and the door still lies, that is our health bug. Fix the door. Do not write a second watchdog.

---

## Locks that stay

- Day opens when the tape starts. Marks are not what make tape processable.
- Remote mark is fallback.
- Alarm rule: if it fires on a healthy ordinary day, the alarm is wrong.
- Spare-exists means a device is present.
- No Pulse writes. No Slack writes. No live `stt_turn`.
- 22 Aug STT door PRD stands. Hear 24 August with Whisper when someone asks.

---

## Acceptance

| # | Pass |
|---|---|
| 1 | Home Office Mini has the agent. Satya/Pruthvi confirm it in the console. |
| 2 | `launchd` job is present and checks at least once a minute. |
| 3 | Close the room tab. Within a minute the URL is open again and `scribe_diff_room` shows listening. |
| 4 | Tape does not auto-start. |
| 5 | Reload path: listener stale, page still up → reload, listener fresh. |
| 6 | No native app shipped. |

---

## What comes back

- Which Minis have the agent.
- The script + plist that shipped (gist or architecture repo, no secrets).
- One Home Office trial: close the tab, door goes listening again, time-to-recover.
- Any Mini that has no agent (name it). Those cannot be saved this way.

IT deploys. Designer does not click the ME console. Kush already has the standup ask to talk to Satya. That is the door in.
