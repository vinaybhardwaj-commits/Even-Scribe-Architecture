# ETA / Even Scribe — VAD + silence gates for STT
## Product Requirements Document (product path)

| | |
|---|---|
| Status | Locked for the builder. Open questions in §8. |
| Date | 14 September 2026 |
| Author | Scribe Designer |
| Audience | Vinay → builder orchestrator. Cut coder briefs from this. Not a ticket list. |
| Repo | Design only. Product work stays in **Even-Transcription-Assistant**. Do not implement here. Do not push to the product repo from this seat. |
| Why now | Whisper on quiet Bench tape still writes speech. Mini will ship **infrastructure** VAD (STT drain, route, emotion shims). whisper.cpp already runs Silero internally. **Callers still treat any decode as speech.** This file is the truth shield for the ETA / Scribe **app + MCP job path**. |

This is the product gate. Mini infra is a different ship. Both must exist. Internal Silero on the Mini is not a product gate.

---

## 0. One-line job

Do not send near-silent audio to Whisper as if it were speech, and do not write `stt_turn` (or any speech cue) from Whisper filler, sticky-decode loops, or near-identical collapse. Mark the window honestly: `silent`, `stt_degraded`, or incomplete.

---

## 1. Problem

Quiet rooms do not come back empty. They come back as words.

### 1.1 What we already measured

The 19 Aug OPD 7 hole walk (`BUILDER-REPORT-OPD7-HOLE-WALK-22-AUG-2026.md`) is the picture. Do not re-walk it to prove this PRD.

- K5 works: HTTP 200 + empty text → one `stt_silence`, `stt_window.complete: true`, zero `stt_turn`.
- Whisper **rarely** returns empty on quiet audio. Of ~eight effectively quiet 15-minute windows, **one** was true `stt_silence`. The other seven were one-to-three filler turns, a token stretched across ten minutes, or a short phrase repeated.
- Two windows were 100% identical filler. One was 49 of 81 turns as a single repeated sentence. Afternoon counts looked busy and were not.
- Downstream cannot tell a one-turn hallucination from a brief real exchange. Fuse, list-turns, and any live cue consumer then treat filler as speech.

That defect is not a K5 bug. K5 records what Whisper admitted. The transcriber refuses to admit silence. The greedy pin reduced loops; it did not remove them.

### 1.2 Sticky-decode (triplets)

On silence and near-silence, Whisper (whisper.cpp included) still **conditions on previous text**. The decoder latches a short n-gram and emits it again: triplets and near-identical runs (`thank you` / `you` / a generic token). Cost and latency go up because the slice was sent. The graph then contains **false speech cues**. Operators and the fuse read a clinic that did not happen.

`condition_on_previous_text=false` is a required decode pin on this path. It is not a substitute for skipping the slice.

### 1.3 Why Mini Silero is not enough

whisper.cpp on Mini already has Silero VAD internally. Product still gates callers because:

1. Internal VAD does not stop every quiet window from producing text.
2. **Callers** (`transcribe_range`, `room_window`, stitch, encounter inputs, any live sink) currently treat non-empty text as speech.
3. Emotion / route / drain shims on Mini can still be fed filler if the app path does not refuse the slice.
4. Repeat-ratio and consecutive-collapse are **app-side** honesty, not a Mini model flag.

Mini ships infrastructure VAD first. This PRD is the app/MCP truth shield so those callers do not launder Whisper filler into the day.

### 1.4 August 22 ruling — supersede, do not forget

`DESIGNER-REPLY-OPD7-HOLE-WALK-22-AUG-2026.md` §1 said: do not add VAD; do not invent a content threshold that rewrites filler as silence; a heuristic that mints silence hides the same defect the bake-off hid.

That ruling was right **for that weekend**: do not paper a walked day after the fact, do not delete loop turns from the 19 Aug graph, do not pretend coverage.

**This PRD supersedes “do not add VAD” for new listens on the product path.** The honesty constraint stays:

- Gate **before** decode on energy / VAD of the asked slice (or of sub-slices after stitch).
- Gate **after** decode on repeat-ratio / near-identical collapse.
- Write a **named marker**, not a tidier day. Readers must still see “we asked, it was silent / degraded,” not a missing window.
- Do **not** rewrite the 19 Aug scratch graph with this heuristic. New asks and new jobs only, unless Vinay names a re-walk.

`stt_silence` remains what K5 made it when Whisper returns empty. This path adds a **pre-Whisper silent skip** and a **post-Whisper degraded** mark. Those are not the same cue. See §4.

---

## 2. Goals / non-goals

### Goals

1. Every STT caller on the ETA / Scribe **app + MCP job path** refuses near-silent slices instead of decoding them as speech.
2. Silent skip writes `silent` (see §4). Degraded decode writes `stt_degraded`. Incomplete / failed asks still write incomplete `stt_window` (K4/K5). Never write `stt_turn` from filler.
3. Whisper decode on this path uses `condition_on_previous_text=false`, plus consecutive / near-identical collapse and a repeat-ratio tripwire.
4. Kill-switch env flags so a bad threshold can be turned off without a deploy guess.
5. Acceptance on three fixtures: silent clip, speech clip, code-mix OPD. Do not use paid engines to prove the gate.

### Non-goals

| Out | Why |
|---|---|
| Mini infrastructure VAD (STT drain, route, emotion shims) | Mini’s ship. This PRD consumes it; it does not specify Mini internals. |
| Replacing whisper.cpp Silero | Keep it. Do not treat it as the product gate. |
| A second STT stack / vendor VAD service | One clip path. Reuse Mini + existing ffmpeg stitch. |
| Rewriting 19 Aug scratch turns | Honesty of that walk stays. Re-walk is a named later ask. |
| Converting old loop `stt_turn` rows into `stt_silence` | That is the papering we forbade. New gates on new asks. |
| Live `room_day` speech writes | Producer types stay blocklisted. Scratch / job writes only, same as today. |
| Flipping `NEXT_PUBLIC_ETA_LIVE_SINK` in prod | Unchanged. If a live sink exists and is on somewhere, it **must still gate** (§3.3). Do not turn it on to test this. |
| Visit mint, fuse rewrite, Pulse, Slack, Gerrit | Unchanged. |
| Emotion / diarize quality | `emotion_window` / `diarize_window` must not run on a slice this gate skipped. They do not get their own VAD design in this file. |
| Clinician UI / kiosk restyle | Out. |

### Split of labour (locked)

| Seat | Owns |
|---|---|
| Mini | Infra VAD on drain / route / emotion shims; whisper.cpp flags including internal Silero. |
| ETA app + MCP jobs | Caller gates: skip, mark, collapse, tripwire, kill-switch. **Truth shield.** |
| Designer | This file. Threshold numbers in §5 are starting locks; retune only with named fixtures and a short report. |

---

## 3. Where gates apply

One function on the product side. Every path below calls it **after** the asked window is resolved to a clip (or to stitched pieces) and **before** a speech cue is written. Do not fork a second “quiet check” per tool.

If Mini already returned `silent` / no-speech on the clip, the app still marks the window and **still does not write turns**. Do not decode “just in case.”

### 3.1 `transcribe_range` (sync tool + job)

Covers:

- MCP `scribe_transcribe_range` (default dry-run stays).
- Job kind `transcribe_range` (`scribe_job_submit`, including `async:true` on the listen tool).

Behaviour:

- Resolve window → stitch/trim as today (asked window, not the five-minute slab).
- **Gate the asked clip** (and each piece if you measure per-piece then AND the result — see open Q1).
- Near-silent: **do not call Whisper** (or ignore text if Mini already decoded). Dry-run returns `silent` + window fields, no `turns`. Write path: §4.1.
- Speech: existing window-as-unit write. Then post-decode collapse + repeat-ratio (§4.3–4.4).
- Window cap ≤ 30 minutes, `room_recording` join refuse, dry default, scratch-only writes: unchanged.

### 3.2 `room_window`

Job kind `room_window` is a listen over a room clock window. Same gate as `transcribe_range`. Same markers. Same “never write turns from filler.”

If `room_window` fans into multiple `transcribe_range` steps, the gate is on **each** child clip, not only the parent job. A parent that mixes silent children and speech children must not flatten filler into one speech blob.

### 3.3 Live sink (if any)

`NEXT_PUBLIC_ETA_LIVE_SINK` stays off in prod unless Vinay already flipped it. This PRD does not flip it.

**If** a live 250 ms (or similar) ear is running in any environment:

- Gate **before** posting any live speech cue / sink event.
- Near-silent frames: drop. Do not emit filler tokens as turns.
- Do not invent a new live cue type in this slice. Drop is enough for live. Markers in §4 are for **windowed** Bench/MCP asks (scratch `stt_*`). Live must not write `stt_turn` on a live `room_day`.

If there is no live sink process, say so in the builder report and skip this row. Do not build a live ear to satisfy the table.

### 3.4 Stitch path

Stitch joins covering pieces, then trims to the asked window. The gate runs on the **trimmed asked clip** that Whisper would see.

- Do not gate only the first piece and then decode the join.
- Do not send a silent join to Whisper because one covering piece had a cough at the edge.
- Sub-slice skip (optional, Q2): if the join is long and only a middle band has energy, you may skip silent heads/tails **without** changing the asked-window write unit. The marker still covers the **asked** window. Do not mint extra windows.

Stitch that is audio-only (no STT) does not need the speech gate. The STT job that consumes the stitch does.

### 3.5 Encounter transcript inputs

Doctor-PWA / encounter lab (`scribe_stt_runs`, encounter pipeline, any path that feeds Whisper or Mini ASR into an encounter transcript):

- Same skip: near-silent media does not become transcript text.
- Same post-decode collapse + repeat-ratio.
- Do not write encounter transcript rows from filler. Mark the run `silent` or `stt_degraded` on the run record you already have. **Do not** create Bench `stt_turn` cues from the PWA encounter path (that coupling stays forbidden).
- Code-mix OPD is an acceptance fixture (§6), not a reason to disable the gate.

### 3.6 Must not run on a skipped slice

If the gate skipped STT, do **not** start `diarize_window` or `emotion_window` on that same clip as if it were speech. Cancel / skip those children with the same silent/degraded reason. Do not bill a second model for a quiet room.

Compare (`scribe_stt_compare`) is observe-only and still must not treat filler as a bake-off winner. Apply the same skip; return `silent` / `degraded` per engine rather than ranking hallucinations.

---

## 4. Behaviour (locked)

Honesty over a tidy day. Named states. No speech cues from filler.

### 4.1 Near-silent slice — skip decode, mark `silent`

When energy / VAD says the asked clip (or the whole window) is below threshold:

| | |
|---|---|
| Whisper | **Do not call** (or discard Mini text if Mini decoded anyway). |
| `stt_turn` | Zero. |
| Marker | One `stt_silence` for the asked window **or** the same K5 silence cue with payload flag `gated: vad` / `reason: near_silent` so readers can tell “Whisper returned empty” from “we never sent it.” Prefer **one type** (`stt_silence`) + `payload.gate` rather than a new cue type. |
| `stt_window` | `complete: true`, `segment_count: 0`, `silent: true`, `gate: vad` (or equivalent payload fields). This was an ask that finished. |
| MCP / job result | `ok: true`, `silent: true`, no turns. Not `failed`. |

K5 empty-transcript silence stays: if the gate is off or the clip passes VAD and Whisper returns 200 empty, still `stt_silence` with `gate: empty_transcript`. Both are quiet rooms. Both are complete. Downstream must not treat either as speech.

### 4.2 `stt_degraded`

Use when the clip **was sent** (or Mini returned text) and post-decode checks say the text is not usable speech:

- Repeat-ratio tripwire (§5.4).
- Consecutive near-identical run that dominates the window (§5.3).
- Classic sticky triplet / loop (same as the hole-walk “loop” report class).

| | |
|---|---|
| `stt_turn` | **Zero.** Do not keep “a few” filler turns “for evidence.” The marker is the evidence. |
| Marker | Completeness cue `stt_window` with `complete: true`, `stt_degraded: true`, `reason` in (`repeat_ratio`, `near_identical`, `sticky_decode`), counts: `segment_count_raw`, `repeat_ratio`. Optional one `stt_silence` is **wrong** here — we do not claim the room was silent; we claim the ear is unusable. Prefer **no** `stt_silence` on degraded. |
| MCP / job | `ok: true`, `stt_degraded: true`, `silent: false`. Not `failed`. |

Thin-but-real speech (1–3 distinct segments on a long window, **not** identical) stays speech. That is the August “thin” class: not a consult, not silence, **not** this gate. Do not fold thin into `stt_degraded`.

### 4.3 Consecutive / near-identical collapse (before the tripwire)

On a window that still looks like speech:

1. Collapse consecutive segments whose normalised text is identical or near-identical (see §5.3) into one turn. Keep first start, last end, one copy of the text.
2. If after collapse the window **still** trips repeat-ratio → §4.2 (`stt_degraded`, zero turns).
3. If collapse leaves real distinct phrases → write those turns as today.

Collapse is a sanitiser for sticky tails on **real** consults (code-mix OPD included). It is not a licence to keep a 49× repeated sentence as one turn of speech. That window is degraded.

### 4.4 Incomplete window (unchanged K4)

Non-200, timeout, stitch fail, time budget: no `stt_silence`, no turns, `stt_window.complete: false`, `stopped_early`. Gate skip is **not** incomplete.

### 4.5 Never write turn cues from silent Whisper filler

Locked:

- No `stt_turn` from a skipped slice.
- No `stt_turn` from a degraded window.
- No live speech cue from a dropped live frame.
- No encounter transcript body from filler.
- `scribe_post_cue` blocklist unchanged. Operators cannot paste filler onto a live day as `stt_turn`.

### 4.6 Payload fields (additive)

On `stt_window` (and on the MCP/job summary):

```
silent?: boolean
stt_degraded?: boolean
gate?: vad | empty_transcript | none
reason?: near_silent | repeat_ratio | near_identical | sticky_decode
segment_count_raw?: number
repeat_ratio?: number
energy_dbfs?: number
vad_speech_ratio?: number
condition_on_previous_text: false
```

Do not put transcript text on the marker. Do not paste transcripts into Slack, chat, or this repo.

`source_ref` / window-as-unit / delete-then-insert for the asked window: unchanged.

---

## 5. Recommended settings (starting locks)

Retune only against the §6 fixtures. Put every number in env (§7). Defaults below are the ship defaults when the gate is **on**.

### 5.1 Energy / VAD (pre-decode)

Measure on the **mono clip Whisper would receive** (post-stitch, post-trim), 16-bit PCM or the Mini’s native analysis — do not invent a third resampler.

| Knob | Default | Meaning |
|---|---|---|
| Speech probability (Silero or Mini-equivalent) | `0.5` | Frame is speech if ≥ this. |
| Speech ratio | `≥ 0.05` of frames in the asked clip | Below → near-silent skip. A 15-minute warehouse quiet with a door slam must still skip. |
| RMS energy | `≤ -40 dBFS` **and** speech ratio below threshold → skip | Energy alone can lie on AC hum; require **both** low speech ratio and low energy, **or** speech ratio alone if Mini VAD is trusted. Locked preference: **speech ratio primary, energy confirmatory.** Skip if `vad_speech_ratio < 0.05`. If VAD unavailable, skip if RMS ≤ -40 dBFS **and** peak ≤ -25 dBFS. |
| Pad | 200 ms | Do not shave the first phoneme on a real consult. Pad is for optional sub-slice trim (Q2), not for changing the asked window. |

Do not require a second ML VAD in Node if Mini already returns `speech_ratio` / `silent` on the clip. **Callers must honour that flag.** If Mini does not yet return it, compute RMS/peak locally and fail open to **skip** when both are below floor (quiet) rather than fail open to **decode** (today’s bug).

Fail-open decode (old behaviour) only when the kill-switch is off (§7).

### 5.2 Whisper decode pin

On every Whisper call from this path (Mini whisper.cpp and any app-side whisper client):

| Knob | Lock |
|---|---|
| `condition_on_previous_text` | **`false`** |
| Greedy / beam 1 / temperature 0 / seed | Keep the existing pin from slice A. |
| Language | One language per file remains a named prior. Code-mix may garble; the gate must not classify code-mix as silence. |

Internal Silero in whisper.cpp may stay on. It does not replace §5.1.

### 5.3 Consecutive / near-identical collapse

Normalise: Unicode NFKC, lower case, strip punctuation, collapse whitespace.

| Knob | Default |
|---|---|
| Identical consecutive | Always collapse. |
| Near-identical | Normalised Levenshtein ratio ≥ 0.92 **or** one string is the other plus a 1–2 token stutter. |
| Max consecutive run collapsed | No cap on collapse; the **tripwire** decides degraded. |

### 5.4 Repeat-ratio tripwire

This is the sanitize note. There is **no separate STT-sanitize folder** in Even-Scribe-Architecture today. Do not write a second PRD that duplicates this section. Fold future sanitize edits here (or a short delta that links here).

The hole-walk loops **are** this tripwire’s fixtures: 100% identical filler; 49/81 one sentence.

Definition (locked):

- Let `N` = segment count **before** collapse (raw Whisper segments).
- Let `M` = count of segments whose normalised text equals the **modal** normalised line.
- `repeat_ratio = M / N` for `N ≥ 4`.
- Trip if `repeat_ratio ≥ 0.60` **or** any consecutive identical run length ≥ `8` **or** (`N ≥ 10` and unique normalised lines ≤ 2).

On trip → §4.2 `stt_degraded`, zero turns.

Do not trip a real consult that repeats a drug name three times. `N ≥ 4` plus 60% modal dominance is the floor that caught 49/81 and 100% filler. If a code-mix consult trips, that is a builder bug in normalisation or a report back to the designer with counts only (no transcript).

### 5.5 What “thin” still means

Unchanged from August: `segment_count` 1–3 on a window ≥ 10 minutes, texts **not** identical → **thin**. Write the turns. Do not mark `stt_degraded`. Do not mark `silent`. Downstream still must not treat thin as a visit. No new scoreboard field required in this slice.

---

## 6. Acceptance tests

No transcripts in the report. Counts, flags, latencies, engine, window.

Fixtures: use Bench tape already on R2 (Home Office test takes, 19 Aug OPD 7 quiet window, a known consult window). Do not mint visits. Scratch or dry-run only.

| # | Name | Setup | Pass |
|---|---|---|---|
| A | Silent clip | Window the walk already treated as true quiet (OPD 7 `12:37–12:52` IST / `07:07Z`, or a known empty Home Office blip). Gate **on**. | Whisper **not** called (or Mini `silent` honoured). `silent: true`. Zero `stt_turn`. `stt_window.complete: true`. Not `failed`. Dry default writes nothing; `dry_run:false` on scratch writes silence marker only. |
| B | Speech clip | Known consult audio (Cardiology ~06:54Z Dibyendu window, or Home Office OSCE piece). Gate **on**. | Turns written (or dry-run lists turns). `silent: false`, `stt_degraded: false`. Distinct phrases survive collapse. Not skipped as silence. |
| C | Code-mix OPD | Indic + English consult window (Sarvam/Whisper as today; **Whisper path must pass**). Gate **on**. | Not classified silent. Not `stt_degraded` unless the clip is actually a loop. Language prior may still garble; garble ≠ skip. Report `language` Whisper picked. |
| D | Filler / loop (sanitize) | A window the hole walk called loop (e.g. 14:37–14:52 IST class), **new ask** on scratch copy, not a rewrite in place of 19 Aug rows unless Vinay says re-walk. | `stt_degraded: true`. Zero new `stt_turn` for that ask. `repeat_ratio` present. |
| E | Kill-switch | Same silent clip as A, gate **off**. | Old behaviour allowed for the switch test only: may decode. Report whether Whisper returned filler. Turn the switch **back on** before leaving the environment. |
| F | Job path | `scribe_job_submit` `kind=transcribe_range` or `room_window` on the silent clip. | Job completes `done`, result `silent`, no turn cues. `diarize_window` / `emotion_window` not started on that clip. |
| G | Closed | No Pulse. No Slack. No live `room_day` `stt_turn`. No transcript in chat. `condition_on_previous_text=false` visible on the Whisper request or Mini log. |

A+B+C are the product bar. D proves we are not repeating August’s “thin filler as speech.” E proves we can abort. F proves MCP jobs, not only the sync tool.

---

## 7. Rollout / kill-switch

Ship **behind flags**, default **on** in preview, **on** in prod only when A–C pass on preview with the same build. One master switch beats six forgotten knobs.

| Env | Default | Job |
|---|---|---|
| `ETA_STT_SILENCE_GATE` | `1` | Master. `0` = today’s decode-everything path (kill-switch). Markers from this PRD are not written when off, except K5 empty-transcript silence which already exists. |
| `ETA_STT_VAD_SPEECH_RATIO` | `0.05` | Skip if below. |
| `ETA_STT_VAD_PROB` | `0.5` | Frame threshold if the app computes VAD. |
| `ETA_STT_ENERGY_DBFS` | `-40` | Confirmatory RMS floor. |
| `ETA_STT_CONDITION_ON_PREVIOUS` | `0` | Must be false/0 on Whisper calls when gate is on. When master is off, still **prefer** false; do not re-enable previous-text to “fix” code-mix without a designer note. |
| `ETA_STT_REPEAT_RATIO` | `0.60` | Tripwire. |
| `ETA_STT_REPEAT_RUN` | `8` | Consecutive identical run trip. |
| `ETA_STT_COLLAPSE` | `1` | Consecutive near-identical collapse. `0` disables collapse only (tripwire may still fire). |

Names may match existing ETA env style (`process.env` already used on the product). If a name collides, keep the collision and document the alias in the builder report. Do not add a second master flag.

Rollout:

1. Preview: flags on. Run §6 A–C (and D if the loop fixture is easy).
2. Prod MCP/jobs: flags on after preview pass. Do not re-walk 19 Aug.
3. Kill: `ETA_STT_SILENCE_GATE=0`. Expect filler again. That is the point of the switch.

Mini infra flags stay Mini’s. Do not overload Mini’s Silero env as the app master switch.

---

## 8. Open questions for the builder

Answer in the report. Do not block the slice on Q2/Q5.

1. **Per-piece vs asked-clip.** After stitch, is Mini VAD available on the trimmed join only, or per covering piece? Lock after you read the Mini ship: product gate is on what Whisper sees (the trimmed join). Per-piece skip of silent heads/tails is allowed only if the asked-window marker still covers the full ask.
2. **Sub-slice decode.** May we send only the energetic middle of a 15-minute window to Whisper while still marking the asked window as mixed silent+speech? **Designer preference: not in v1.** v1 is all-skip or all-decode for the asked window (plus collapse/tripwire). Mixed interior speech in a warehouse hour stays a later slice so we do not invent extra write units.
3. **Mini `silent` payload.** What exact field will drain/route return (`silent`, `speech_ratio`, `energy_dbfs`)? Honour it by name. If it ships after this app slice, RMS fallback in §5.1 holds.
4. **Live sink presence.** Is any process actually consuming `NEXT_PUBLIC_ETA_LIVE_SINK` in preview? If no, §3.3 is N/A. If yes, show a dropped-frame counter, not a transcript.
5. **`stt_degraded` on `stt_window` vs a new cue type.** This PRD prefers payload flags on `stt_window` and zero turns, no new type. If the uniqueness key or list-turns summary cannot show degraded without a type, propose `type=stt_degraded` as one cue covering the window (same family, same blocklist, same `source=replay`). Do not do both.
6. **Encounter run schema.** Where does `silent` / `stt_degraded` land on `scribe_stt_runs` without quoting transcript? Name the column or JSON field.
7. **Threshold retune.** If A skips a real consult or C trips degraded, do not silently loosen. Send counts (speech_ratio, dBFS, N, M, unique lines) and stop.

---

## 9. Locks that stay closed

- Window-as-unit. Delete-then-insert for the asked window. Marker INSERT / `ON CONFLICT DO UPDATE`.
- Dry-run default on `scribe_transcribe_range`. Scratch writes only for turns/silence.
- Producer types blocklisted on live `scribe_post_cue`.
- `stt_turn` does not mint or close visits. Arm A stays the visit writer.
- Join refused while any room is recording. Windows ≤ 30 minutes.
- No Pulse writes. No Slack writes. No raw R2 keys. No transcripts in this repo.
- Decoder pin stays greedy. This PRD adds `condition_on_previous_text=false` and caller VAD; it does not add a second stack.
- Compare still does not write.
- 19 Aug OPD 7 “heard” picture stays; this gate is for **new** asks.

---

## 10. What comes back to the designer

- Product sha (preview / prod).
- Flag values actually set.
- §6 table: pass/fail per row. Whisper called: yes/no. Turn counts. `silent` / `stt_degraded`. Repeat-ratio on D.
- Whether Mini returned a silent flag or the app used RMS fallback.
- Whether live sink existed.
- Any open question you had to pick (especially Q5).

Do not send transcripts.

---

## 11. Read with this file (do not duplicate)

| File | Why |
|---|---|
| [BUILDER-REPORT-OPD7-HOLE-WALK-22-AUG-2026.md](./BUILDER-REPORT-OPD7-HOLE-WALK-22-AUG-2026.md) | Filler vs true silence; loop counts. The problem’s evidence. |
| [DESIGNER-REPLY-OPD7-HOLE-WALK-22-AUG-2026.md](./DESIGNER-REPLY-OPD7-HOLE-WALK-22-AUG-2026.md) | Weekend “do not add VAD” — **superseded for new product listens** by this PRD; honesty constraint kept. |
| [DESIGNER-REPLY-SILENCE-AND-K4-22-AUG-2026.md](./DESIGNER-REPLY-SILENCE-AND-K4-22-AUG-2026.md) | `empty_transcript` is silence, not fail. Still true. |
| [OPERATOR-STT-AND-BRAIN-DOOR-PRD-22-AUG-2026.md](./OPERATOR-STT-AND-BRAIN-DOOR-PRD-22-AUG-2026.md) | Door, engines, window-as-unit. This file does not rebuild S0–S4. |
| [SPEECH-TURNS-SCRATCH-BRIEF-22-AUG-2026.md](./SPEECH-TURNS-SCRATCH-BRIEF-22-AUG-2026.md) | `stt_turn` / `stt_silence` shape. |
| [DESIGNER-REPLY-SLICE-A-WINDOW-KEY-22-AUG-2026.md](./DESIGNER-REPLY-SLICE-A-WINDOW-KEY-22-AUG-2026.md) | Write unit = asked window; refuse partial. |

Sanitize / repeat-ratio: **this PRD §5.3–5.4** is the note. No second copy.
