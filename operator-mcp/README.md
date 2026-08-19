# Builder note — Operator MCP

19 Aug 2026, rev 3e. From Vinay / Scribe Designer.

**This folder is for the orchestrator.** Design lives in `Even-Scribe-Architecture`. The designer does not push to the product repo. The designer does not implement. The coder does not treat the PRD as a ticket list.

## Seats

| Seat | Job |
|---|---|
| Orchestrator | Read the PRD + inventory. Use the Scribe tree and the Pulse monorepo (read-only) to write coder instructions. |
| Coder | Build the slice you scoped, on `feat/operator-mcp` in **Even-Transcription-Assistant**. Report done / blocked to you. |
| Designer | Reviews what you report. Revises the PRD here. |

Loop: designer → this repo → orchestrator → coder → orchestrator → designer.

## Read in this folder

1. [DESIGNER-REPLY-TO-BUILDER-19-AUG-2026.md](./DESIGNER-REPLY-TO-BUILDER-19-AUG-2026.md) — S1–S3 accepted, remount locked.
2. [EVEN-SCRIBE-MCP-PRD-19-AUG-2026.md](./EVEN-SCRIBE-MCP-PRD-19-AUG-2026.md) — locks. §8.5 is remount.
3. [SCRIBE-MCP-STACK-INVENTORY-19-AUG-2026.md](./SCRIBE-MCP-STACK-INVENTORY-19-AUG-2026.md) — what existed at `b907bc50`.
4. [EVEN-SCRIBE-FUSE-PRD-19-AUG-2026.md](./EVEN-SCRIBE-FUSE-PRD-19-AUG-2026.md) — visit-constructor. Next slices.
5. [BUILDER-REPORT-S1-S3-19-AUG-2026.md](./BUILDER-REPORT-S1-S3-19-AUG-2026.md) — the report this reply answers.

A coder brief names the slice, the real files, the §18 rows that close it, and what is out (production kiosk, Pulse writes, Slack).

Do not merge `feat/operator-mcp` until Vinay says the 19 Aug tapes are safe.
