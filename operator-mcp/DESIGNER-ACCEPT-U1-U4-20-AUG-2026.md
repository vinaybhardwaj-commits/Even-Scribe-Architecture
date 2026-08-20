# Designer acceptance — MCP U1–U4
20 Aug 2026, evening. Scribe Designer. For the orchestrator.

Read `BUILDER-REPORT-U1-U4-20-AUG-2026.md`. Production `main` `99484f9`. V authorised the merge.

## Call

**U1–U4 accepted.** The upgrade request is closed.

I re-ran U3 from this chat against Cardiology 19 Aug: three sessions, `end_time_disagrees` only on `bs_j9wgfa33` (tape end `05:30:28Z` vs stored `09:56:54Z`). The tool found the crash without being told.

U2 (12-minute Ankit window, four pieces, one 719,997 ms clip) and U1 (dry-run identical, still one live cue `cue_95z4avnv`) accepted on your measurements. U4 accepted on `bs_wy5yjj7a`.

## Settled from this report

1. **U4 cannot rescue 19 Aug `bs_8temrdqh`.** No loss events that day. Correct. It protects tapes from the instrumentation onward. Do not special-case that session.
2. **Backup in replay (fuse §12 Q3).** The recording's own mic events decide. Explicit `source` always wins. No backup → nothing invented. Closed.
3. **Determinism** compares tool output, not the transport envelope. Closed.
4. **No clinician surface** until a visit exists. Fuse PRD §10 stands. Do not design a doctor screen in this programme.
5. **Clustered reloads still drop the in-flight segment (D4).** Leave it. Cumulative cost is a note, not a change. Revisit only if a clinic day is actually F5-stormed.

Cloudflare container-library stream bug is infra. Out of this PRD. Do not block fuse on it.

## Scratch graph (fuse §12 Q2) — confirmed

Same tables. First migration of this programme is allowed.

| | |
|---|---|
| Cues | `source: replay` (already forced by `scribe_post_cue`) |
| `room_day` | boolean `scratch` (or equivalent). Replay writes only land on `scratch=true` rows until I say otherwise. |
| Live 19 Aug graph | Untouched. Still empty of replay. |
| Idempotent | Natural key session + type + at. Re-replay does not duplicate. |
| After I review a scratch day report | A second pass may write the live `room_day`. Not before. |

That is fuse PRD §11 slice 2. Warehouse join is 3. Fuse on scratch is 4. The day report you owe (marks vs warehouse vs visits vs tape) is 5, after 2–4.

## Next

Cut slice 2 (scratch write + this migration). Preview is fine; you are already on `main` with V's word — still do not run the scratch migration in a way that rewrites 19 Aug live rows.

Send the scratch dry-run-then-write report here. I will review before warehouse join posts into it.
