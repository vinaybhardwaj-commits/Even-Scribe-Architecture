# Designer reply — K4 is live, silence can fire
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-NOTE-K4-SHIPPED-22-AUG-2026.md`. You are right: the K3 stamp was right about the file and behind on the build. I am not asking anyone to rebuild K4.

## K4 — accepted as shipped

`fc24776` matches the list I named. Marker upsert with no DELETE. Marker id `cue_j8r7keqc` survived the 06:00Z re-run (`deleted: 144`, not 145). Counts split, and `dropped` vs `already_existed` as two bugs is better than the spec. One DELETE in the brain, on `cue`. `visit` and `speaker_cluster` unused. No extra migration.

The stamp's "next" line is spent.

## Silence — `empty_transcript` is silence, not a fail

A 200 with no text is a quiet window. Do not abort the tool.

On `empty_transcript`:

1. Write `stt_silence` for that asked window (`source = replay`, `at` = window start, payload as already specified).
2. Upsert `stt_window` with `complete: true`, `segment_count: 0`. Same INSERT-only path as K4. This was an ask that finished.
3. Zero `stt_turn` rows.

A non-200 (`http_*` or any other error) is a failed ask: no `stt_silence`, no turns, upsert `stt_window` `complete: false` with `stopped_early`. If that upsert also fails, the MCP response is the only record. Say so.

Do not treat `empty_transcript` as `failed`. Do not invent speech. Do not mint a visit.

That is the walk-blocker. Ship it, then walk the rest of `04:36:59Z → 10:39:35Z` in asked windows ≤ 30 minutes. Then the rest of the finished sessions. Send the hole-walk report before treating 19 Aug as heard. Same dry default, same scratch guard. Live days untouched.

The four-window sample (three with speech, one quiet at lunch) is noted. I do not have `ETA-SPEECH-TURNS-HOLE-SAMPLE-REPORT-22-AUG-2026.md` in this thread. Park it on this repo if you want it next to the walk.

## Monday file — I do not have it

`ETA-MONDAY-READINESS-AND-OPEN-RULINGS-22-AUG-2026.md` is not on this repo and was not attached. I will not guess the five questions. Send that file.

Consent copy and a second microphone cannot be invented after Monday starts. I still do not write consent copy. Counsel does.

## Admin Bench monitor — heard, not expanded

V asked, V approved, you are building. I am not blocking an operator tool he asked for two days before a live day.

Boundary, so it does not become the room screen I forbade:

- Admin cookie only. No clinician. No kiosk. No Pulse UI.
- Observe: listeners, tape state, last mark, last window asked, `stt_window.complete`. Not a visit editor. Not a fuse writer.
- No auto-start from this page. No live `room_day` write. `scribe_post_cue` blocklist still holds.
- JSON behind the MCP door remains the artefact. This page does not replace `scribe_fuse_report`.

If the design crosses that, send it before it ships. I do not need to re-approve what V already approved inside this fence.

## Unchanged

No live `room_day`. No B or C. No run id on `visit`. No transcripts in Slack, chat, or this repo. `stt_turn` does not mint or close a visit. Arm A stays the visit writer. Window-as-unit stays. Dibyendu = 1. No Pulse. No Gerrit.
