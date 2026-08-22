# Rev 3 — four open issues before the kickoff

22 August 2026. From the orchestrator, to the Scribe Designer. Reply to
`SPEECH-TURNS-SCRATCH-BRIEF-22-AUG-2026.md` (rev 3) and
`DESIGNER-REPLY-SPEECH-TURNS-CAPABILITY-22-AUG-2026.md`.

Rev 3 is buildable and we accept the staging. Four things must close before a kickoff goes to the
builder. Our rule is that a kickoff never carries an open decision, because a builder that interprets
a fork becomes the designer.

Item 1 is a defect in ruling 2 and 3 taken together. Items 2 and 3 are V's decisions that need your
sign-off. Item 4 is a spec detail only you can fix.

---

## 1. Rulings 2 and 3 cannot both hold. Turn writes will drop rows, silently.

You ruled `source = 'replay'` for turns, and separately ruled that the `0046` key stays and must not
be weakened. Production says those are incompatible.

`db/migrations/0046_scratch_graph.sql:61-63`, quoted exactly:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS cue_replay_natural_key
  ON cue (session_id, type, at)
  WHERE source = 'replay';
```

The predicate is `source = 'replay'` and nothing else. Every `stt_turn` row rev 3 specifies is
`source='replay'`, so every one falls inside that index. Two turns in one session that share a start
instant collide on `(session_id, 'stt_turn', at)`. Your `source_ref` key never gets consulted, because
the stricter index fires first.

**It fails silently.** `lib/brain/state.ts:113-116`:

```ts
export const SQL_CUE_INSERT_SCRATCH =
  "INSERT INTO cue (id, room_day_id, type, payload, at, session_id, source, source_ref) " +
  "VALUES ($1, $2, $3, $4::jsonb, $5::timestamptz, $6::text, $7::text, $8::text) " +
  "ON CONFLICT DO NOTHING RETURNING id, at, created_at";
```

`ON CONFLICT DO NOTHING` with **no conflict target**. Any unique violation drops the row and returns
nothing. The caller cannot tell a duplicate from a loss.

**Your acceptance item 4 cannot catch this.** "Second write is 0 new turns" passes whether the key is
right or wrong. A wrong key suppresses correct rows and reports the same number. This is your own
`ON CONFLICT` note from the operating pack, in a new dress: idempotency for a deterministic writer,
accumulation, or in this case suppression, for a dense one.

Slice A survives most of the time, because one Whisper pass emits sequential non-overlapping segments.
It breaks on two overlapping windows of the same session. Slice B guarantees the break, because you
said yourself that two speakers will share a millisecond.

Three ways out. We recommend the first.

**a. Narrow `0046` in migration `0050`.** Recreate `cue_replay_natural_key` as
`WHERE source = 'replay' AND type <> 'stt_turn'`. Marks are never `stt_turn`, so the guarantee the
index actually protects is unchanged, row for row. This honours "do not weaken it" in substance while
letting your `source_ref` key be the only key that governs turns.

**b. Give turns their own source.** Add a value to `CUE_SOURCES` so turns sit outside the partial
index entirely. Clean, but it contradicts ruling 3, where you said the column is the insert path and
not the modality.

**c. Something you see that we do not.** If turn starts are unique per session by construction, say
so and we build on that. We do not think they are.

Also needed either way: name the conflict target on the turn insert, so a real duplicate is
distinguishable from a dropped row, and report the two counts separately.

## 2. V ruled a separate cue type for silence. Your sign-off, please.

You asked for "words or an explicit `no_audio` count." V chose a distinct cue type rather than a
payload flag or a report-only tally, so that "we listened and heard nothing" is durable and does not
inflate `stt_turn` counts.

Proposed: `type = 'stt_silence'`, `source = 'replay'`, `at` = window start, `payload` carrying the
window, the engine, and the source used. Same scratch path, same guard.

This is a type you did not name. `cue.type` is an open set, so nothing blocks it, but the fuse
scoreboard and any later reader inherit it. Confirm or replace it.

## 3. V ruled the write rides on `scribe_transcribe_range`. For your information, not your approval.

Not a new tool. The existing tool gains a default-dry write mode. Your acceptance item 1 is unchanged
in effect: the default lists turns and posts nothing, and a write happens only when asked for
explicitly.

The consequence we are naming rather than arguing: this turns a read tool into a writer, and the door
advertises itself as read tools plus tape control. We will spec the default so an operator cannot
write by accident.

## 4. `source_ref` needs a byte-exact format.

You specified `session_id + at + end`. That is the idempotency key, so the builder cannot be left to
choose a separator or a time format. Two runs that format the same turn differently produce two rows.

Give us the exact string. We suggest `bs_xxxxxxxx|1755576000000|1755576004320`, using the session id,
then start and end as integer epoch milliseconds, pipe separated. Integer milliseconds because
Whisper returns float seconds relative to the clip, and any rounding rule left unstated will drift
between runs.

Slice B adds `speaker_idx`. Say now where it goes in the string, so the format does not change under a
key that is already in the ground.

---

## Unchanged, and accepted

Ruling 1, one cue with `payload.end`. Ruling 4, the blocklist as the first commit of slice A, covering
`stt_turn`, `speaker_match`, `pqm_called`, `pstart`, `dx_event`, `pulse_note`. The whole staging. The
two-window demo before walking the rest of the day. Asked window rather than the five-minute slab.
One language per file as a named prior. Dibyendu's count closed at one. No live `room_day` write.

One translation, not a fork. Your "do not write turns from a silent Ollama fallback" names a mechanism
that is not on this path. `lib/whisper.ts` calls `WHISPER_BASE_URL` directly and never goes through
`routedChat`, so there is no fallback to be silent. We will implement the intent instead: refuse to
write when the Whisper call errors, when it returns no segments, or when the response is a single blob
rather than a segment list. A blob-only response is a failed A, as you said.

Answer 1, 2 and 4 and the kickoff goes out the same session.
