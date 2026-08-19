# Orchestrator recommendations — after S1–S3
20 Aug 2026, 04:40 IST. Scribe Designer. For the builder orchestrator.

This updates the call after your S1–S3 report. Design lives in `Even-Scribe-Architecture`. Product work stays on `feat/operator-mcp` in `Even-Transcription-Assistant`. Designer does not push there.

---

## What changed in my head

The Operator MCP door is no longer the bottleneck. S1–S3 are accepted. Cues flow, tape is remotely startable, extract-by-clock works, pin is a cue.

What the 19 Aug day actually taught:

1. **The kiosk still splits the day on remount.** That is now the highest-leverage product bug. It also spoiled your two remaining §18 rows.
2. **Voice will not bind this corpus.** Far-field doctor vs phone centroid is 0.55–0.62. Treat it as a prior. Marks + warehouse clocks are the identity path.
3. **Warehouse join is real.** PQM / pstart / Rx URL already land inside tape windows. That is enough to name visits on replay. Do not wait for a better ear.
4. **K-A / K-B on `main` are inventory.** Dual-mic + janitor stay. Remount must use the same stall window, not fight it.
5. **The missing piece is the fuse**, run first as replay → scratch against this 13 h tape. Not a live 250 ms ear.

---

## Do this, in this order

Cut coder briefs yourself. Cite files from the product tree. Do not ask the designer for a patch list.

### 1. Remount (product, `feat/operator-mcp`)

Lock: MCP PRD §8.5 and [DESIGNER-REPLY-TO-BUILDER-19-AUG-2026.md](./DESIGNER-REPLY-TO-BUILDER-19-AUG-2026.md).

- Tab remount + session still owned + last chunk inside the 10 min stall window → **resume** same `session_id`.
- Stalled or ended → do not resume. Janitor ends the zombie. Next Start is new.
- Then re-run the two spoiled §18 rows: idempotent start, clean stop.

This is the only kiosk behaviour I want changed before fuse work.

### 2. Echo `cue_id` on `scribe_mark_consult`

Nit. When the cue lands, return it. Same slice as remount if that is cheaper.

### 3. Replay dry-run

`scribe_replay_session` on one 19 Aug session (Ankit `bs_xvntaugh` first). Cue list only. Post nothing.

### 4. Scratch write

Same replay, `source:replay`, scratch `room_day`. Do not touch a live 19 Aug graph.

### 5. Warehouse join for that IST day

One-shot. Direct PG if the RO role exists, else a Metabase fixture. Post `pqm_called` / `pstart` / `dx_event` / `pulse_note` at real clocks. Marks already on the tape become `consult_mark` cues.

### 6. Fuse on scratch

[EVEN-SCRIBE-FUSE-PRD-19-AUG-2026.md](./EVEN-SCRIBE-FUSE-PRD-19-AUG-2026.md).

Gemini Flash. Debounce §15A. Advisory lock already there. Replace the echo on `POST /cues`.

### 7. Day report back here

Marks vs warehouse vs visits vs tape. That is what I review. Then we decide whether a second pass may write the live `room_day`.

---

## Do not

- Merge `feat/operator-mcp` to `main` until Vinay says the tapes are safe.
- Flip `NEXT_PUBLIC_ETA_LIVE_SINK` in prod.
- Change the 1–2 s poll interval. Fine at two rooms.
- Spend a slice on `/api/admin/r2-cors-fix`. Infra. Token lacks bucket-settings.
- Bind visits with cosine. 0.78 stays banned. 0.55–0.62 is not a doctor ID.
- Silent `UPDATE visit` from `scribe_pin_visit`. Pin is a cue.
- Auto-start anything from warehouse.
- Write Pulse. Post Slack. Push this design into the product repo.
- Ask the designer to implement, launch a coding agent, or open a product PR.

---

## What I will review

- Remount: F5 keeps the same session; a 5 h zombie does not resume.
- Idempotent start + clean stop green on preview.
- Dry-run posts nothing.
- Scratch day report: every mark window that contains a pstart / `called_at` has a visit with that `individual_uid`. Hole-shaped days are one visit. Voice-only binds stay low confidence.

Send that report back to Vinay → this designer. I will revise the fuse PRD if the corpus moves a lock.
