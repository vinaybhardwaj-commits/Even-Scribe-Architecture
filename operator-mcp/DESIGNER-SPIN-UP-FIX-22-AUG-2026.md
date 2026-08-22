# Designer issues — spin-up prompt is stale
22 Aug 2026. Scribe Designer. For the orchestrator / builder.

Read `SPIN-UP-PROMPT-22-AUG-2026.md` and `OPERATOR-SURFACE-22-AUG-2026.md`. V asked that you fix the spin-up. I am not rewriting either file. These are the issues. A new thread that pastes the current prompt will treat speech turns as not-yet-started and treat migrations after `0049` as unexpected.

The §11 close still stands. Hard rules still stand. This is a currency fix, not a new programme.

---

## Spin-up — must change

**1. Check 3 is wrong.** "Highest should be `0049_visit_ambiguity`" is false. `0050` and `0051` are live (turn key, two turns sharing `at`). The window-unit fix will add at least `stt_window` to the type lists. Write: highest is **`0049` or later**. If you see only `0049`, stop and say so — that is a rolled-back or unread prod, not the expected state.

**2. Check 1 is a floor, not the picture.** `9010ba8` or later is fine as a minimum. Last reported prod for slice A was `ded0451`. Do not tell a new thread that `9010ba8` is current. Record the sha they actually get.

**3. "Where things stand" stops too early.** Fuse §11 is closed. That is not the live work. Add, after the fuse paragraph:

- Speech turns slice A is **in flight**. First listen proved we can hear Dibyendu's consult. Whisper is not a deterministic writer. Write unit is now the **asked window** (delete-then-insert). Refuse a partial window; record `stt_window` completeness. Pin decoder if Mini accepts it.
- **Do not analyse** the 77 Cardiology `stt_turn` rows on `rd_scratch_bh6jtq4t_20260819`. Delete them before the fixed writer runs.
- OPD 7's 35 rows at 04:02Z may stay. That window is complete. **It is not the hole.**
- The hole window (`04:36:59Z → 10:39:35Z` on OPD 7) is still unrun. That is the open product question. Then one complete Cardiology window around 06:54Z.
- Do not walk the rest of the day until I have that two-window report.

Rulings: `DESIGNER-REPLY-SLICE-A-WINDOW-KEY-22-AUG-2026.md`.

**4. Missing operating locks that already shipped.**

- `scribe_transcribe_range` writes, default **dry**. An operator must not write by accident.
- `scribe_post_cue` blocklist on a live day: `stt_turn`, `stt_silence`, `stt_window`, `speaker_match`, `pqm_called`, `pstart`, `dx_event`, `pulse_note`. Producer types only via the scratch path.
- `stt_turn` does not mint or close a visit. Arm A stays the visit writer.
- No transcripts in Slack, chat, or the architecture repo.

**5. Slice 6 wording.** "All six slices are done" is easy to misread. Five delivered, six **skipped**. Keep the skip.

**6. Keep, unchanged.** LLM health check (all six `gemini:`). Do not write the live `room_day`. Do not re-run B or C. No run id on `visit`. No clinician page / room screen. `LAST_MARK_WINDOW_MS` = 45 min. The `ON CONFLICT` gotcha (now proven on Whisper). MCP tool-list cache. The owed list (token, Neon roles, misspelled secret, CDS model, day two, consent, `.env.example`). Ask session focus before multi-step.

---

## Operator surface — stamp only, do not rewrite

`OPERATOR-SURFACE-22-AUG-2026.md` stays the §11 reference. Do not turn it into a daily log. Add a one-line banner at the top:

> Snapshot at `9010ba8` / `0049`. Speech turns superseded Part 5 item 1. Current spin-up is `SPIN-UP-PROMPT-22-AUG-2026.md`. Do not paste this file as a carry-over.

If you touch Part 2 at all: `scribe_transcribe_range` is default-dry write, and `scribe_post_cue` cannot post the blocklisted types on a live day. "Sources: mcp, replay" as written is the live-write hazard we closed.

---

## Done when

A new thread can paste the spin-up and will: accept sha later than `9010ba8`, accept migrations later than `0049`, treat speech turns as the live work, refuse to read the 77 Cardiology rows as the consult, and not write the live day.
