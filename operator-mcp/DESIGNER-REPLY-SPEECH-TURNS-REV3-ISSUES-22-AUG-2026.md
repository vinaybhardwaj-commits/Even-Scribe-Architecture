# Designer reply — rev 3 open issues
22 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-REV3-OPEN-ISSUES-22-AUG-2026.md`. Four forks. All closed. Kickoff may go.

## 1. Narrow 0046. Accept (a).

Rulings 2 and 3 cannot both stand against `cue_replay_natural_key`. You are right. `ON CONFLICT DO NOTHING` with no target would drop a real turn and look like a duplicate.

**Migration `0050`:** recreate `cue_replay_natural_key` as

```sql
(session_id, type, at)
WHERE source = 'replay'
  AND type NOT IN ('stt_turn', 'stt_silence', 'speaker_match')
```

Marks stay protected, row for row. That is “do not weaken it.” Turns (and later matches) are governed only by the `source_ref` key.

Add the turn unique index:

```sql
UNIQUE (source_ref, type)
WHERE source = 'replay'
  AND type IN ('stt_turn', 'stt_silence', 'speaker_match')
```

Turn insert **names that target**. Report three counts: written, already existed, dropped (a drop that is not a named conflict is a bug). Acceptance item 4 is rewritten: second write is 0 written, N already existed, 0 dropped.

Do not take (b). Source stays `replay`. Do not take (c). Starts are not unique by construction.

## 2. `stt_silence` — accept.

`type = 'stt_silence'`, `source = 'replay'`, `at` = window start, `payload` = `{ end, engine, source_used, window }`. Same scratch path. Does not inflate `stt_turn`. “We listened and heard nothing” is a cue, not a report-only tally.

Add `stt_silence` to the `scribe_post_cue` blocklist in the first commit of A.

## 3. Write on `scribe_transcribe_range` — noted.

Default stays dry. Write only when asked. An operator must not write by accident. I am not asking for a new tool.

## 4. `source_ref` byte-exact.

Always four fields, pipe-separated, no spaces:

```
{session_id}|{start_ms}|{end_ms}|{speaker}
```

- `session_id` is the Bench id (`bs_…`).
- `start_ms` and `end_ms` are **integer epoch milliseconds**, floored. Convert Whisper’s float seconds (relative to the clip) onto the asked window’s absolute start, then `Math.floor`. Do not round. Do not use ISO strings.
- `speaker` is `-` in slice A and on `stt_silence`. Slice B writes the integer `speaker_idx` in that same slot (`0`, `1`, …). The format does not gain a fifth field.

Examples:

```
bs_xvntaugh|1755576000000|1755576004320|-
bs_xvntaugh|1755576000000|1755576004320|0
```

Silence uses the **window** start and end in the same shape, `speaker` `-`.

Two runs that hear the same segment must produce the identical string. If they do not, the key is wrong.

## Unchanged

One cue, `payload.end`. Blocklist first. Two-window demo first. Asked window, not the slab. One language per file. Dibyendu = 1. No live `room_day`. Whisper path has no `routedChat`; refuse write on error, no segments, or a blob.

Rev 3 brief stands except where this note overrides (0050, `stt_silence`, `source_ref` format, named conflict counts).
