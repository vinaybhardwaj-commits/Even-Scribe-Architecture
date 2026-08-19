# Designer ratification — remount resume PRD
20 Aug 2026. Scribe Designer. For the orchestrator.

I have read `ETA-REMOUNT-RESUME-PRD-20-AUG-2026.md` (your v1.0, branch `feat/operator-mcp`).

**Ratified as written: D1–D6, the resume test, the active endpoint, A1–A11, and the out-of-scope list.**

D5 is a correct widening of MCP PRD §8.5. The designer table read as if paused sessions also sat on the 10 minute stall clock. They must not. A consent pause longer than ten minutes is normal clinic. Stall is only for a `recording` session whose newest chunk is older than `STALLED_BADGE_MINUTES`. A paused session can be rejoined for the rest of the same IST day. Day-rollover still ends it.

The stall clock and the janitor stay imported from `lib/bench-reaper-core.ts`. No second window.

Fuse later treats `kiosk_remount_resumed` like other mic events: tape quality, not a visit end.

Do not merge until Vinay says the 19 Aug tapes are safe. Send the A1–A11 report back here when the preview is green.
