# Operator MCP — build report to the Scribe Designer
**19 Aug 2026, from the orchestrator. Covers PRD rev 3d slices S1–S3 on `feat/operator-mcp`, plus what the two-room pilot forced onto `main` today. No secrets in this doc; shareable.**

## What shipped on the branch (not merged; preview-verified against production data)

| Slice | Sha | Contents | Acceptance |
|---|---|---|---|
| S1 door + reads | `5ecde16` | `/api/mcp` (hand-rolled JSON-RPC, zero new deps), constant-time `SCRIBE_MCP_TOKEN`, 23 read tools, `GET /api/brain/rooms/[id]/cues`, session list filters + marks | PASS — incl. byte-identity of the unfiltered sessions GET vs prod |
| S2 bus + tape control | `b77ef48` → rebased `b2f6384` | `bench_command` + `bench_listener` (0044), kiosk 1.5 s poll, remote start/pause/resume/stop, listeners in `scribe_health` | Core PASS: remote start made a real tape; dark room → `kiosk_not_listening`, no orphan row; pause/resume acked. Two §18 rows (idempotent start, clean stop) owed a clean re-run after a mid-test tab reload spoiled the first attempt |
| S3 writes + clock-time audio | `a8273f8` | `scribe_post_cue` (source forced `mcp`) · `scribe_mark_consult` (durable-first, your C1 pattern) · `scribe_pin_visit` (cue only — `visit` proven untouched) · `scribe_extract_audio` (IST clock → presigned single-chunk clip + offsets; multi-chunk returns the covering chunks honestly) · `scribe_transcribe_range` (Mini Whisper, real text verified) · `scribe_list_commands` | PASS |

33 tools live. Your §19 open questions were all ratified with your defaults (V, 19 Aug): one token v1 · Mini ffmpeg multi-chunk v1.1 · get_visit v1.1 · last-poll-wins · watch by standing poll · replay dry-run + scratch. Full decisions log (D1–D10 + builder-flag rulings A1–A9): `ETA-OPERATOR-MCP-VERIFICATION-AND-DECISIONS-19-AUG-2026.md`.

## What the pilot forced onto `main` today (outside your PRD, V-ratified R1–R12)

The two rooms hit both failure classes live: a tab crash left a session "recording" for 5 h, and a dead external mic recorded silence invisibly. Shipped to `main`: **K-B** dual-mic capture (backup stream from the machine's built-in mic, same session, `bench_chunk.source`, migration 0045) + primary-mic failsafe (detach listener + silence watchdog + auto-resurrect, yank-test proven) and **K-A** session janitor (auto-end at last-chunk time, 30 min / day-rollover; red `stalled` chip at 10 min). Your Bench surfaces gained: per-source chunk arrays, mic-story `bench_event` kinds (`mic_primary_lost/restored/…`) rendered in timeline.md, and mic badges. The branch is rebased onto all of it.

## Defects + design questions for you

1. **Remount semantics (design call needed):** if the kiosk page reloads while its session is paused/live, the current code ends the old session and starts a new one (crash-recovery semantics). When the kiosk still owns that session, we think it should RESUME it — same session id, same tape. Your call on the rule.
2. `/api/admin/r2-cors-fix` has never worked (`put_cors_failed: Access Denied` — token lacks bucket-settings scope; proven tonight, likely since May). Fix the token, or delete the route per its own comment.
3. Nit: `scribe_mark_consult` doesn't echo `cue_id` even when the cue lands.
4. Ops note, not a change request: 1–2 s kiosk polling ≈ 43–86k function invocations/day/room. Fine at 2 rooms; a bill at 10. Worth a line in a future rev.

## The pilot corpus and the ask

~13 h of verified tape across both rooms, with consult marks all day. We've scoped a five-layer validation: tape → marks → voice-cluster consistency (far-field caution: even the doctor scores 0.55–0.62 vs his phone centroid) → warehouse truth (PQM table joins proven: patient, check-in, prescription-start times + prescription URL land inside the tape windows) → **the fuse**. Replay tooling (`scribe_replay_session`, dry-run + scratch per your default) is next on our side.

**The ask: the fuse design round.** Everything upstream of it now exists — cues flow, evidence is listable, replay is speccable, ground truth is in hand. The visit-constructor (what reads cues and writes the picture) is the piece only you can design. Send the design when ready and we'll build against this corpus.
