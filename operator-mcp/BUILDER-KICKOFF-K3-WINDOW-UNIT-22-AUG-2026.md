# Slice A, kickoff K3 — the window is the write unit

22 August 2026. Orchestrator. Base: `main` `ded0451`, prod migrations through `0051`.

Whisper is not a deterministic writer. Two runs of the same clip returned 162 and 165 segments, so a
key built on segment boundaries accumulates instead of deduplicating. The designer has changed the
write unit from the turn to the window. This kickoff builds that.

Everything here is settled. Do not reopen a decision. Flag anything the spec does not cover.

Amends `ETA-SPEECH-TURNS-SLICE-A-PRD-22-AUG-2026-v1.0.md` and kickoff K2. Both stand except where this
file overrides.

---

## The five changes

1. Window-as-unit write: delete then insert, in one transaction.
2. Batch the insert. One transaction, not one POST per cue.
3. A completeness cue, `stt_window`.
4. Refuse a window you cannot finish. Roll back rather than commit a partial.
5. Pin the Whisper decoder.

## 1. Window-as-unit

A turn is still one cue. `at` is the start, `payload.end` is the end. `source_ref` is still
`{session_id}|{start_ms}|{end_ms}|{speaker}` and is still the **within-write** key, so two turns that
share a start both land. `0051` proved that case in production and it stays.

`source_ref` is **not** the cross-run key. Segmentation is Whisper's opinion about a window, and an
opinion gets replaced, not merged.

The write unit is `(session_id, asked_window_start_ms, asked_window_end_ms)` on the scratch room-day.

In one transaction:

1. **Delete** every `stt_turn`, `stt_silence` and `stt_window` row for that pair. Match on
   `session_id` and on the **asked window in the payload**, never on Whisper's segment times:

```sql
DELETE FROM cue
 WHERE room_day_id = $1
   AND session_id  = $2
   AND source      = 'replay'
   AND type IN ('stt_turn', 'stt_silence', 'stt_window')
   AND (payload->'window'->>'start_ms')::bigint = $3
   AND (payload->'window'->>'end_ms')::bigint   = $4
```

2. **Insert** the new set.

Report four counts: `deleted`, `written`, `already_existed`, `dropped`. After a delete
`already_existed` should be 0. A second run of the same window may write a different segment count.
That is a replace and it is correct, not a failure. A drop that is not a named within-write conflict
is still a bug.

Source stays `replay`. Do not add a value to `CUE_SOURCES`.

## 2. Batch the insert — this is not optional

The current path posts one cue per HTTP request. Measured in production on the Cardiology window:
row `created_at` timestamps ran `05:03:21.819` to `05:04:06.122`, which is 44.3 seconds for 71 rows,
**624 ms per row**.

At that rate a 165-segment window costs 103 seconds of inserts alone, before the join and the
transcription add roughly 60 more. The tool's own cap is 115 seconds. So raising the time budget
cannot make a six-minute window finish. Batching can.

Add a batch mode to the existing route, `POST /api/brain/cues`. Do not build a second route and do not
write to the database from the MCP tool directly. The scratch guard must not be duplicated.

The batch path reuses, unchanged:

- `withRoomDayLock(roomDayId, …)`
- the re-read of `scratch` **inside** the lock
- the 409 `not_a_scratch_day` refusal

Delete and insert both run inside that one lock and one transaction.

## 3. The completeness cue

```
type        stt_window
source      replay
at          asked window start
source_ref  {session_id}|{window_start_ms}|{window_end_ms}|window
payload     { end, engine, source_used, complete, segment_count, language, stopped_early? }
```

`complete: true` rides in the same transaction as the turns, or as the `stt_silence`.

`complete: false` carries `stopped_early` and the `segment_count` Whisper returned, and **zero**
`stt_turn` rows exist for that window. A reader can then tell "we asked and could not finish" from "we
never asked."

Delete-then-insert applies to `stt_window` for the pair, exactly as for turns.

## 4. Refuse a partial

A silent partial is worse than a refusal. If the turn set cannot be written whole, **roll back the turn
insert** and commit only the `stt_window` row with `complete: false`. Never leave 71 of 162.

The 30-minute ask cap stays.

## 5. Pin the decoder

Ask whisper.cpp for greedy decoding, beam size 1, temperature 0 and a fixed seed. Check whether the
Mini endpoint accepts those parameters. **If it does not, flag it in the report. Do not guess and do
not build a second STT stack.**

This reduces churn. It does not make the writer deterministic, which is why the window is the unit.

## 6. Migration `0052_stt_window_type`

`stt_window` joins the same family as `stt_turn` and `stt_silence`, so it must appear in **both**
predicates. Both need a drop and a recreate.

```sql
DROP INDEX IF EXISTS cue_replay_natural_key;

CREATE UNIQUE INDEX IF NOT EXISTS cue_replay_natural_key
  ON cue (session_id, type, at)
  WHERE source = 'replay'
    AND type NOT IN ('stt_turn', 'stt_silence', 'stt_window', 'speaker_match');

DROP INDEX IF EXISTS cue_turn_natural_key;

CREATE UNIQUE INDEX IF NOT EXISTS cue_turn_natural_key
  ON cue (source_ref, type)
  WHERE source = 'replay'
    AND type IN ('stt_turn', 'stt_silence', 'stt_window', 'speaker_match');

INSERT INTO schema_migrations (version, name)
VALUES (52, '0052_stt_window_type')
ON CONFLICT DO NOTHING;
```

`SQL_CUE_INSERT_TURN`'s `ON CONFLICT … WHERE` clause must be updated to match the new predicate
**character for character**. A test already asserts that equality — keep it passing rather than
relaxing it.

Add `stt_window` to the `scribe_post_cue` blocklist, making eight types.

## 7. Cleanup

Delete the 77 `stt_turn` rows on `rd_scratch_bh6jtq4t_20260819` from the asked window
`06:52:00Z–06:58:00Z`. They are two disagreeing opinions of an unfinished window.

Note that the new writer's own delete step covers them when that window is re-run, since all 77 carry
the same `payload.window`. A separate delete is only needed if they must be gone sooner. Provide a way
to do it that does not need a new tool on the door.

**Leave OPD 7's 35 rows at 04:02Z.** That window is complete and stable, and it is the standing proof
that a deterministic window is replace-safe. It is not the hole window and must not be read as one.

---

## File contract

Create or edit ONLY:

- `db/migrations/0052_stt_window_type.sql` (new)
- `app/api/brain/cues/route.ts` — add the batch path, leave the live branch untouched
- `lib/brain/state.ts` — add statements, change none
- `lib/mcp/tools/bench.ts`
- `lib/mcp/tools/brain.ts` — the blocklist
- `lib/mcp/tools/fuse-report.ts` — count `stt_window` and surface `complete`
- `lib/whisper.ts` — decoder parameters only
- tests under `tests/unit/`

Never touch:

- `SQL_CUE_INSERT_SCRATCH`, and migrations `0050` and `0051`
- the **live** branch of `app/api/brain/cues/route.ts`, the one with no `room_day_id`
- `REPLAY_KINDS` and `buildReplayCues`
- the arms, `visit`, `LAST_MARK_WINDOW_MS`
- `lib/diarize.ts`, `lib/stt/eta-router.ts`, the encounter pipeline
- deploy config and the dependency manifest

## Commit order

1. Migration `0052` and the blocklist.
2. The batch path on the route, with the guard and lock reused.
3. Delete-then-insert in the tool, with the four counts.
4. `stt_window`, and the refuse-partial rollback.
5. The decoder pin, or a flag saying the endpoint refused it.
6. The `scribe_fuse_report` counters for `stt_window` and completeness.

## Same rules as before

No live `room_day` write. The tool defaults dry. No new `CUE_SOURCES` value. No second STT stack. No
clinician page and no room screen. No Pulse, no Slack, no Gerrit. Turn text never goes into chat, Slack
or the repo.

You have no live database, so every query you write is inferred. List each one verbatim in your
report. An error degrades to an empty result or a no-op, never a 500 and never a wrong write.

Gate before pushing: tests green, typecheck clean, production build green. Push to `main`.

Report: sha, gate results, inferred SQL verbatim, whether whisper.cpp accepted the decoder parameters,
deviations, and confirmation that `0052` records itself.
