# Designer reply — slice A two-window report
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-SLICE-A-TWO-WINDOWS-22-AUG-2026.md`. Prod `c93078f`, migrations through `0053`.

## Call

**Both complete windows are accepted.** The replace worked. The 77 bad Cardiology rows are gone. The refuse-partial rule held on the failed first hole attempt. Decoder pin stays, and its value is higher than we claimed: temperature sampling invented a wrong clinical term.

**The hole finding is accepted.** OPD 7 `04:36:59Z → 10:39:35Z` is not an empty room. One six-minute probe at 06:00Z is continuous speech and a full dermatology consult. The warehouse has nothing in that window. The remaining five hours and fifty-four minutes are now the question.

`stt_turn` still does not mint or close a visit. Arm A stays the visit writer. This is evidence that Pulse missed care, not a new scratch visit.

Do not write the live `room_day`. Live OPD 7 stays one cue. Live Cardiology stays three kiosk marks.

## Marker — INSERT-only. Accept the recommendation.

§2 failed for a structural reason: the fallback reused `replace_window`, so it needed the same DELETE the primary write needed. When the grant was missing, both posts died and the day stayed silent.

**`stt_window` must not go through `replace_window`.** Write it INSERT / `ON CONFLICT (source_ref, type) DO UPDATE` only. No DELETE. The marker must not inherit the verbs the turn write needs. Do not add a second MCP tool.

On a failed window:

1. Do not commit turn rows (unchanged).
2. Upsert one `stt_window` with `complete: false`, `stopped_early`, `segment_count`. Zero `stt_turn` / `stt_silence` for that window.
3. If that upsert also fails, the MCP response is the only record. Say so. Do not pretend the day recorded the ask.

On a successful window, upsert `complete: true` in the same transaction as the turns, still without DELETE on the marker.

`source_ref` stays `{session_id}|{window_start_ms}|{window_end_ms}|window`.

Ship this before walking more of the day. A second structural failure must leave a marker.

## Counts — split `dropped` from `failed`.

`dropped` is only a named within-write conflict. A transport, permission, or incomplete write is `failed`, with a reason. Do not assign the turn count to `dropped` on the failure path. 144 drops would have been read as 144 key collisions.

Report: `deleted / written / already_existed / dropped / failed`.

## Grants — `0053` on `cue` is accepted. Do not use the rest.

DELETE on `cue` is required for window replace. DELETE on `visit` and `speaker_cluster` was not requested. Do not DELETE those tables unless I ask. `room` stays SELECT-only. Do not revoke `0053` for this slice.

A migration remains the only owner-privileged path. Keep grants there.

## Window-as-unit stays.

Instability is a property of the messy audio, not of Whisper. The pin does not let us relax the write unit. Cardiology needed replace. The clean OPD 7 window did not. Keep replace for every window.

## Next — walk the hole, then the rest of the finished day.

After the marker upsert and the count split:

1. Walk the rest of OPD 7 `04:36:59Z → 10:39:35Z` in asked windows ≤ 30 minutes. Words or `stt_silence`. No new visit.
2. Then the rest of the finished sessions on both scratch days (skip `bs_8temrdqh` by name).
3. Send a hole-walk report before treating 19 Aug as heard.

Same dry default. Same scratch guard. No transcripts in Slack, chat, or this repo. Dibyendu = 1. No Pulse. No Gerrit.

The scoreboard may later count warehouse-silence-with-speech. Do not add that field in this walk. The report is enough.
