# Operator door — STT engines and the brain
## Product Requirements Document

| | |
|---|---|
| Status | For the builder orchestrator |
| Date | 22 August 2026 |
| Author | Scribe Designer |
| Audience | Builder **orchestrator**. Not a ticket list for the coder. |
| Why now | Tonight the operator heard Home Office through the door. Whisper on the Mini is the only engine the listen tool will run. Deepgram, Sarvam, ElevenLabs, and IndicConformer all answered a health ping. The old PWA bake-off last ran in July. The operator can see the brain and can post a cue, but cannot pick an engine, cannot compare engines on one tape window, cannot change routing, and cannot list room-days. Grow the door. Do not rebuild K3–K5. |

This is an upgrade of the **operator door**. It is not a new brain, not a new recorder, not a second fuse, and not a clinician page.

Cut coder briefs yourself from this. Product work stays in `Even-Transcription-Assistant`. Design lives in `Even-Scribe-Architecture`. Designer does not push to the product repo.

---

## 1. One-line job

Let the operator hear any Bench window through any working STT engine, compare those engines on the same clip, change how Scribe routes speech, and read the brain the same way the fuse does — without writing a live clinic day and without replacing Gemini-fast.

---

## 2. What is already live (do not rebuild)

Measured 22 Aug ~19:23 IST against production door `99484f9`.

### 2.1 Hear

`scribe_transcribe_range` is **Whisper only** (`engine` enum is one value). One finished piece works even while another room is recording. Join across pieces is refused by name (`room_recording`) until the day ends. Window-as-unit, decoder pin, `stt_silence` on a 200 empty, refuse-partial, marker upsert (K4/K5) stay. Default is dry-run. Write replaces the asked window on **scratch**.

Tonight: Home Office `bs_mry2rj9a` (Cook history video) and `bs_7etnyqz5` (OSCE chest-pain demo) both heard via Whisper. The second reply was ~41 KB because the turns array is always attached. That is a door defect. Fix it in S0.

### 2.2 Raw audio

`scribe_extract_audio` and `scribe_get_recording` already give short-lived links. Never inline bytes. Join still refused while any room is recording. Keep that.

### 2.3 Engine registry (health ping, 22 Aug 13:53Z)

| id | Name | Enabled | Health | Notes |
|---|---|---|---|---|
| `deepgram` | Deepgram nova-3-medical | yes | 782 ms | live English route |
| `whisper` | Whisper large-v3-turbo (Mac Mini) | yes | 185 ms | note English route; only hear engine today |
| `sarvam` | Sarvam Saaras v3 | yes | 28 ms | live Indic + note Indic |
| `elevenlabs` | ElevenLabs Scribe v2 | yes | 352 ms | ASR, not on the routing matrix |
| `elevenlabs_scribe` | ElevenLabs → Even Note | yes | 345 ms | `tiers: scribe`, not a hear engine |
| `ekascribe` | EkaScribe v2 | **no** | 2 ms | disabled; refuse by name |
| `even_pipeline` | Even pipeline (ASR → LLM note) | yes | virtual | not a hear engine |
| `indicconformer` | IndicConformer-600M (Mac Mini) | yes | 270 ms | note / Indic |
| `indicconformer_scribe` | IndicConformer → Even Note | yes | 269 ms | `tiers: scribe`, not a hear engine |

Routing (unchanged since 31 May 2026):

- live × english → `deepgram`
- live × indic → `sarvam`
- note × english → `whisper`
- note × indic → `sarvam`

`scribe_stt_health`, `scribe_list_stt_engines`, `scribe_stt_routing`, `scribe_list_stt_runs` are **read**. There is no write. `scribe_list_stt_runs` is the PWA encounter lab, last row 17 Jul. It is not Bench tape.

### 2.4 Brain (already on the door)

Read: `scribe_get_state`, `scribe_list_cues`, `scribe_get_clusters`, `scribe_fuse_report`, `scribe_get_encounter` (PWA), traces.

Write, gated: `scribe_post_cue` (live day; producer types blocklisted), `scribe_pin_visit` (evidence, never a visit UPDATE), `scribe_fuse_run` (scratch only, Arm A is `rules`).

Replay: `scribe_replay_session` dry-run; `scribe_replay_write` to scratch, marks and mic only, no speech turns.

**Missing:** list room-days (live + scratch). Read visits without the scoreboard. List `stt_turn` in a clock window without dragging the full transcribe payload. No engine picker. No routing write.

---

## 3. Locks that stay closed

- Supervisor, not a second fuse. No `scribe_brain_ask` that mints visits. No silent `visit` UPDATE. Pin stays evidence.
- No Pulse writes. No Slack writes. No raw SQL. No raw R2 keys in chat or this repo.
- No live `room_day` write for `stt_turn` / `stt_silence` / `stt_window` / `speaker_match`. Producer types stay on the `scribe_post_cue` blocklist.
- `stt_turn` does not mint or close visits. Arm A (`rules`) stays the visit writer. Do not re-run arms B or C.
- Window-as-unit stays. Delete-then-insert for the **chosen** engine's turns on that asked window. Marker is INSERT / `ON CONFLICT DO UPDATE` only. `empty_transcript` (200) is `stt_silence`. Non-200 is a failed ask. Do not convert hallucination to silence.
- `source_ref` stays four fields: `{session_id}|{start_ms}|{end_ms}|{speaker}`. No fifth field for engine. Compare is observe-only. Only one engine writes a window.
- `payload.engine` is the engine that actually ran. Fail closed if the adapter silently falls back (Ollama, a different vendor, an empty stub). Same spirit as `provider_not_gemini` on the fuse.
- Join refused while any room is recording. Windows ≤ 30 minutes.
- Transcripts are clinical speech. `scribe_list_cues` stays summary-only unless asked. Do not paste transcripts into Slack, chat, or this repo.
- Do not restyle the kiosk. Do not enroll voices this weekend. Do not re-load 52 doctors onto a room. Monday join is (c).
- Home Office 22 Aug is a test tape (YouTube). Do not mint visits from it.
- Consent copy is counsel's.

---

## 4. Slices (this order)

### S0 — Hear returns text, not a 41 KB turn dump

Tonight's second listen wrote a dump the operator could not keep. The words were fine. The payload was not.

| | |
|---|---|
| Default | `scribe_transcribe_range` returns `text`, `language`, `audio_seconds`, `engine`, `source_used`, `chunk_idx` / covering pieces, `turn_counts`, window / completeness fields. **No `turns` array.** |
| `include_turns` | `false` by default. `true` attaches the array. Write-path still builds turns server-side; it does not need to echo them. |
| Same | dry-run default, window-as-unit, Whisper still works with no other change. |

Acceptance: transcribe one Home Office piece (or any 5-minute chunk) with the default. Response is small enough to keep in chat tooling (a few KB). `include_turns:true` still returns the array.

Do this first. Everything below is unusable if the reply cannot be read.

### S1 — Hear with a named engine

Grow `scribe_transcribe_range.engine` from `{whisper}` to every **enabled, non-virtual, `tiers` includes `asr`** adapter:

`whisper` | `deepgram` | `sarvam` | `elevenlabs` | `indicconformer`

| | |
|---|---|
| Default | `whisper`. Omit = today's behaviour. |
| Disabled | `ekascribe` (or any `enabled:false`) → `engine_disabled`. Do not call it. |
| Not a hear engine | `even_pipeline`, `*_scribe` note wrappers → `engine_not_asr`. |
| Fail closed | If the named engine is unhealthy, or the adapter returns another provider, write nothing and say so (`engine_unhealthy` / `engine_fallback`). |
| Turns | Same `stt_turn` shape. `payload.engine` is the named engine, not a hard-coded `whisper`. |
| Decode | Whisper keeps the pinned decoder (greedy, beam 1, temp 0). Other vendors use their documented default. Do not invent a second decoder stack on the Mini. |
| Live tape | One finished piece: allowed (already true). Join: still `room_recording`. |

Acceptance: same Home Office window, `engine=deepgram` and `engine=sarvam`, both dry-run. Each returns text + `engine` + latency. `engine=ekascribe` is `engine_disabled`. Default with no engine field is still Whisper.

### S2 — Compare engines on one window (observe only)

The operator asked to order the APIs. Health is not a bake-off.

New tool: `scribe_stt_compare`.

| | |
|---|---|
| Input | Same window as transcribe (`session_id` or room + IST day, `start`, `end`, `source`). Optional `engines[]`. Omit = every enabled ASR engine that passed the last health ping. |
| Work | Resolve the window **once**. One audio clip. Fan-out to each engine. Do not extract N times. |
| Output | Per engine: `ok`, `engine`, `text` (not turns), `language`, `audio_seconds`, `latency_ms`, `error`. Plus the window and `source_used`. |
| Write | **Never.** No `dry_run:false`. Compare does not touch `cue`. If the operator wants turns stored, they call transcribe with one named engine and `dry_run:false` onto scratch. |
| Caps | Same 30-minute and `room_recording` join rules. One piece while a room is recording is allowed. |
| Cost | Paid engines are on. Say so in the response (`is_paid`). Do not silently skip Deepgram to save money. |

Acceptance: compare the 19:08–19:13 IST Home Office piece (or the 19:02–19:07 piece if the live take is gone) across Whisper, Deepgram, Sarvam. Three texts. Zero new cues on that room-day.

This is not the July PWA encounter lab. Do not hang it off `encounter_id`.

### S3 — Routing and enable, with a dry default

Granular control over **how Scribe uses** the APIs, not only how the operator hears.

| Tool | Job |
|---|---|
| `scribe_stt_routing` | Already live. Keep. |
| `scribe_stt_set_routing` | Set one cell: `stage` ∈ live\|note, `language_bucket` ∈ english\|indic, `engine_id`. |
| `scribe_stt_set_engine` | Set `enabled` and/or `fanout_enabled` on one registry row. |

| | |
|---|---|
| Default | `dry_run: true`. Return what would change. Write only when asked. |
| Refuse | Engine missing, disabled, unhealthy, or not capable of that stage (`stages` on the registry). `engine_not_asr` if someone points `live` at a scribe-tier wrapper. |
| Audit | Each real write is a row the operator can see later (who / when / before / after). Reuse the existing admin audit if it already logs STT lab. Do not invent a second log. |
| Monday | Do **not** change live routing as part of this slice's self-test. The 31 May matrix stays until Vinay names a new cell. Acceptance is dry-run on a preview, plus one write on preview that is reverted. |
| Out | No new engine adapters. No Eka re-enable. No cost column work. |

Acceptance: dry-run `scribe_stt_set_routing` live×english → `elevenlabs` returns the current cell (`deepgram`) and the proposed cell, writes nothing. `scribe_stt_routing` is unchanged. A write on preview flips and a second write flips back.

### S4 — Brain reads the door still lacks

Do not add a second fuse. Add the three reads the operator keeps assembling by hand.

| Tool | Returns |
|---|---|
| `scribe_list_room_days` | Live and scratch days. Filters: room, `ist_date`, `scratch` true\|false\|all (default live-only, same spirit as `scribe_list_rooms`). Each row: `room_day_id`, room, date, scratch flag, visit count, cue count, last cue at, last cue type. No identity. No payloads. |
| `scribe_list_visits` | Visits on one `room_day_id` (scratch or live). Phase, opened/closed clocks, bind state, arm. `individual_uid` only with `include_identity`. No note text. |
| `scribe_list_turns` | `stt_turn` + `stt_silence` + `stt_window` on one room-day in a clock window. Default summary only (`at`, `end`, `engine`, `speaker`, first 80 chars). Full text only with `include_text`. Limit default 100, cap 500. This is how the operator walks a hole without re-running Whisper. |

`scribe_get_state` stays the now-picture. `scribe_fuse_report` stays the scoreboard. Do not merge them.

Acceptance: list room-days for 19 Aug names the two scratch days (`rd_scratch_qyzghzaf_20260819`, `rd_scratch_bh6jtq4t_20260819`) when `scratch` is included. `scribe_list_turns` on the OPD 7 scratch hole window returns the already-written turns without calling an STT API.

---

## 5. What this is not

- Not slice B (clusters, `speaker_match`, enroll). Voice is still a weak prior.
- Not content matching. That is the Monday visit-candidate producer, already ruled.
- Not the live 250 ms ear. Not a second recorder. Not `NEXT_PUBLIC_ETA_LIVE_SINK`.
- Not a note, Rx, disposition, or clinician page.
- Not a rewrite of `scribe_transcribe_range`'s window-as-unit or K5 walk rules.
- Not a 52-doctor warehouse dump. Not Pulse `doctor_opd_rooms`.
- Not admin Bench live-monitor work (already heard, already inside the observe fence).

---

## 6. Engine contract (so the orchestrator does not guess)

A **hear engine** is a registry row with `enabled=true`, `virtual=false`, `tiers` contains `asr`, and `has_adapter=true`.

The adapter must accept the same clip the Mini already builds for Whisper (the asked window, primary unless the session's own mic events say primary was lost). It must return text. Segments are optional.

| If the adapter returns… | Then |
|---|---|
| Segments with start/end | One `stt_turn` per segment (write path only). |
| One blob, no segments | One `stt_turn` covering the asked window. Named as `blob`. Do not invent fake segment times. Whisper must keep returning segments — a blob from Whisper is still a failed A. |
| HTTP 200, empty text | `stt_silence` on the write path. Compare path: `ok:true`, `text:""`, `empty:true`. |
| Non-200 or timeout | `ok:false`, `error` set. Write path: refuse-partial, nothing committed. |

`payload` on a written turn:

```
{ end, text, engine, source_used, window, language, blob?: true }
```

`engine` is the id (`deepgram`, not a display name).

---

## 7. Why compare must not write

`source_ref` is unique on `(source_ref, type)` for replay turns. Two engines on the same timestamps would collide or silently `already_existed`. Adding engine as a fifth field would break slice A keys already on the 19 Aug scratch days.

So: one writer per window. Compare is a report. The operator picks a winner and transcribes with `dry_run:false` if they want it on scratch.

Do not re-key 19 Aug.

---

## 8. Home Office tonight (fixture, not a clinic)

| Session | IST | What it is |
|---|---|---|
| `bs_8rbamfdn` | 18:16, ~6 s | blip |
| `bs_mry2rj9a` | 19:00–19:07 | Cook / Hawaii video. Remount ate the first 90 s. Two marks. |
| `bs_7etnyqz5` | 19:08– (was live) | OSCE chest-pain history (Anas Ali / Zain). |

Use these for S0–S2 if they are still on R2. Do not fuse them. Do not mint visits. Do not write the live Home Office `room_day`.

19 Aug OPD 7 / Cardiology remain the clinic fixtures for S4.

---

## 9. Acceptance (one page)

| # | Slice | Pass |
|---|---|---|
| 0 | S0 | Default transcribe of a 5-minute piece is a small payload: text + counts, no `turns`. |
| 1 | S1 | `engine=deepgram` and `engine=sarvam` dry-run on the same window both return text. `engine=ekascribe` is `engine_disabled`. Omit engine = Whisper. |
| 2 | S2 | Compare on that window returns ≥3 engines, zero cues written. |
| 3 | S1 write | `dry_run:false` with a named engine on a **scratch** day writes turns with `payload.engine` set. Live day still 409 / blocklist. Second write is 0 written, N already existed. |
| 4 | S3 | Routing dry-run proposes a change and writes nothing. Preview write + revert. Production 31 May matrix untouched unless Vinay said so. |
| 5 | S4 | `scribe_list_room_days` sees 19 Aug scratch. `scribe_list_turns` reads existing OPD 7 hole turns without a new STT call. |
| 6 | Closed | No Pulse write. No Slack. No visit mint. No live `stt_turn`. No new kiosk chrome. |

---

## 10. What comes back to the designer

- Commit + which production / preview sha.
- Which acceptance rows pass.
- Per-engine: did the adapter return segments or a blob (name it).
- Any engine that failed closed, with the error it actually returned (not a guess).
- Whether S0 changed the write path (it must not).

Do not send transcripts. Send engine, window, latency, ok/error, turn counts.

---

## 11. Orchestrator notes

- Reuse `lib/stt` adapters and the Mini clip path already used by `scribe_transcribe_range`. Do not add a second ffmpeg stack.
- Do not pull `lib/stt/eta-router.ts` onto the Bench listen path as a silent default. The operator **names** the engine. Routing (S3) is the product default for the PWA live/note stages, not a surprise inside transcribe.
- Do not create a `stt_turn` table. Cue type stays.
- Do not open Eka.
- Do not implement a `scribe_brain_ask`. If you think the operator needs a question-answering tool, that is a product lock — send it back.
- K3 is shipped and superseded. K4/K5 window rules stay. This PRD does not reopen them.
