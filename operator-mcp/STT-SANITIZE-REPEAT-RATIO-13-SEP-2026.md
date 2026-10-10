# STT sanitize — Whisper repetition loops
## Design lock (not a coder ticket list)

| | |
|---|---|
| Status | Locked for the orchestrator |
| Date | 13 September 2026 |
| Author | Scribe Designer |
| Audience | Builder **orchestrator**. Product work stays in `Even-Transcription-Assistant`. |
| Why now | Room-tape drain through Mini Whisper (`whisper.llmvinayminihome.uk`) produced looping phrases (often 3×). Cause is Whisper sticky decode / hallucination — worse on silence, noise, and code-mix — **not** multiple Whisper installs. Drain uses one inference URL. Consecutive identical segments already sit in `verbose_json` before stitch. Sample: ~44% of nonempty jobs had consecutive identical segments. |

This is the later slice named on 22 Aug (`DESIGNER-REPLY-OPD7-HOLE-WALK-22-AUG-2026.md` §1). That weekend walk stays as written: do not re-walk 19 Aug to mint `stt_silence` from filler. **New drain, stitch, fuse, and encounter isolation follow this lock.**

Do not paste transcripts into Slack, chat, or this repo.

---

## 1. Stance

STT is an **evidence producer with known failure modes**, not a trusted oracle.

Pipeline: **gate → decode → sanitize → score**. Fuse and encounter isolation only consume windows that pass.

Raw Whisper text is lab / progress evidence. It is never the encounter transcript.

---

## 2. Confirmed cause (do not re-debug)

| | |
|---|---|
| What | Sticky decode / hallucination. Phrases loop, often three times. |
| Where | Consecutive identical segments in `verbose_json` **before** stitch. |
| What it is not | Extra Whisper installs. Duplicate drain workers writing the same clip. Stitch inventing repeats. |
| Worse on | Silence, noise, code-mix. |

One inference URL. Treat loops as an engine defect, then filter.

---

## 3. Prevention layers (this order)

### 3.1 Don’t send bad audio

VAD / energy gate **before** STT. Skip near-silent slices. Emit a silence marker **without** calling Whisper.

This is not the 22 Aug ban on converting a **non-empty hallucinated** 200 into `stt_silence`. Empty-enough **audio** never reaches the Mini. A looping non-empty decode still stays speech in the graph until sanitize + score; it does not become silence.

### 3.2 Infer settings

- `condition_on_previous_text=false` — do not feed the last loop back in.
- Anti-repeat / no-speech threshold on.
- Prefer short, clean windows. Do not reinforce a sticky prefix across a long slab.

Pinned decoder (greedy, beam 1, temp 0) still stands. This layer does not invent a second Mini stack.

### 3.3 Mandatory post-filter

Never ship raw Whisper text as the encounter transcript.

- Collapse consecutive identical / near-identical **segments**.
- Collapse in-segment **phrase loops**.
- Raw stays in progress / lab evidence only.

Sanitize in `stt_drain` **before write**. Session-id stitch consumes **repaired** text, not the looping `verbose_json`.

### 3.4 Quality tripwires

Keep empty-rate and chars/sec. Add **repeat-ratio**.

If repeat-ratio is over threshold → `stt_degraded`. Do **not** fuse that window as gold speech. Optional second engine or trim. Do not silently swap engines (same fail-closed spirit as `engine_fallback` / `provider_not_gemini`).

Repeat-ratio is a score, not a visit writer. `stt_turn` still does not mint or close a visit. Arm A stays the visit writer.

### 3.5 Product rule

**Encounter transcript** = filtered STT × diarize labels × Chart / PQM clocks.

Brain cues are never minted from unfiltered looping text.

---

## 4. Ops (brief)

Existing Mini drain outs: batch-repair with the same post-process. Do not re-decode the whole corpus unless a repaired file still fails score.

New jobs: sanitize in `stt_drain` before write. Stitch reads repaired text.

Do not put looping copy on Slack, the operator door default payload, or this repo.

---

## 5. What this is not

- Not a second Whisper install, a second ffmpeg stack, or a Mini rewrite.
- Not converting hallucination to `stt_silence` (K5 empty-200 rule stays).
- Not a live `room_day` write. Not a visit mint. Not arms B or C.
- Not work in this architecture repo. Coder briefs land on `Even-Transcription-Assistant`.
- Not a dump of sample phrases. The 44% figure and the 3× pattern are enough.

---

## 6. What comes back to the designer

- Where sanitize runs (`stt_drain` before write — confirm).
- Repeat-ratio definition (consecutive identical segments, plus in-segment phrase loops) and the threshold used.
- Count of jobs marked `stt_degraded` vs repaired vs still looping after repair.
- Confirmation stitch consumes repaired text.
- Confirmation fuse / encounter isolation refuse unfiltered windows.

Do not send transcripts. Send job counts, ratios, and pass/fail.
