# Designer reply — Operator MCP S1–S3
19 Aug 2026 (filed 20 Aug IST). From Scribe Designer. For the orchestrator.

This is the reply to `Operator MCP — build report to the Scribe Designer` (S1–S3 on `feat/operator-mcp`).

## Accepted

| Slice | Call |
|---|---|
| S1 door + reads (`5ecde16`) | Accepted. |
| S2 bus + tape (`b2f6384`) | Core accepted. Two §18 rows still owed a clean re-run: idempotent start, clean stop (tab reload spoiled the first attempt). |
| S3 writes + clock extract (`a8273f8`) | Accepted. Independent cues, pin as cue (`visit` untouched), IST extract, Whisper on a range. |
| §19 defaults | Ratified as written (one token, Mini ffmpeg v1.1, get_visit v1.1, last-poll-wins, watch by poll, replay dry-run + scratch). |
| K-A janitor + K-B dual-mic on `main` | Outside the MCP PRD. V-ratified. Absorbed as inventory. Branch rebase is correct. |

33 tools. Do not merge `feat/operator-mcp` until Vinay says the 19 Aug tapes are safe.

## Remount (the call you asked for)

If the kiosk page reloads while a session is live or paused:

| Situation | Rule |
|---|---|
| This room still owns a `recording` or `paused` session, last chunk **inside** the stall window (not yet the 10 min red `stalled` chip) | **Resume.** Same `session_id`, same tape, keep appending (primary + backup). Restore pause UI if paused. Do not POST a new session row. |
| Session is `stalled` or `ended`, or last chunk older than the stall window | **Do not resume.** Janitor ends the zombie. Next Start is a new session. |
| Tab is gone | Listener drops. MCP start → `kiosk_not_listening` until a tab is back. Then apply the two rows above. |
| MCP start while already recording on a listening room | Still idempotent: return the live session, no second tape. |

F5 must not split the day tape. A 5-hour crash must not keep appending to a dead tape. K-A’s 10 min chip / 30 min janitor is the stall clock.

This is MCP PRD §8.5 (rev 3e).

## The other three

1. **`scribe_mark_consult` must echo `cue_id`** when the cue lands. Coder nit.
2. **Poll cost** — note only. v1 stays 1–2 s at two rooms. Revisit at ten.
3. **`/api/admin/r2-cors-fix`** — infra. Token lacks bucket-settings. Not this PRD. Do not delete until Vinay says.

## Next: the fuse

The visit-constructor is [EVEN-SCRIBE-FUSE-PRD-19-AUG-2026.md](./EVEN-SCRIBE-FUSE-PRD-19-AUG-2026.md).

First run is **replay dry-run + scratch** against the 19 Aug corpus (~13 h, Ankit OPD 7 `bs_xvntaugh`, Dibyendu Cardiology). Not a live 250 ms ear. Far-field voice (0.55–0.62 vs phone centroid) is a prior, not a binder.

Orchestrator cuts the coder slices in that PRD §11. Reports come back here.

## Do not

- Ask the designer to push to `Even-Transcription-Assistant`. Design lives in `Even-Scribe-Architecture`.
- Merge the MCP branch onto the live kiosk until Vinay says the tapes are safe.
- Treat `operator_pin` as a silent `UPDATE visit`.
- Write Pulse. Post Slack.
