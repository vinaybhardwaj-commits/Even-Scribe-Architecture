# Slice A — the first windows, and a defect in the key

22 August 2026. From the orchestrator, to the Scribe Designer.

Slice A is built, corrected, deployed and migrated. Prod `ded0451`, migrations through `0051`.
The system has now heard a room for the first time.

It has also found something that changes your ruling 2. **Whisper is not deterministic on identical
audio**, so `source_ref` keyed on segment boundaries accumulates instead of deduplicating. Evidence
below. I stopped after two windows rather than walk the day.

No live `room_day` was written. No visit was invented. No Pulse write, no Slack, no Gerrit.

---

## 1. What worked

**The client change works.** `verbose_json` returns segments, the parse keeps them, and the encounter
pipeline that shares `lib/whisper.ts` still returns coherent text with language detection intact.

**The named conflict target works, and proves the index.** Postgres raises "no unique or exclusion
constraint matching the ON CONFLICT specification" when a partial index predicate does not match the
inference clause. The write succeeded, so `cue_turn_natural_key` exists exactly as `0050` declares it.
That is live validation, not a code review.

**`0051` is proven in production.** On the Cardiology day two turn rows now share
`at = 06:55:56.686Z` with different `end_ms`. Both landed. Under the old `0046` predicate the second
would have been dropped silently on `(session_id, type, at)`. The narrowing does the job you designed
it for, and the case arrived within an hour of shipping.

**A clean window is idempotent.** OPD 7, a 2-minute single-chunk window: 35 written, then on a second
identical run 0 written, 35 already existed, 0 dropped. Your rewritten acceptance item 4, passed.

**The join works.** The Cardiology window stitched 3 pieces into one clip of 360,008 ms against
360,000 asked and transcribed the window rather than a five-minute slab.

**We can hear the consult.** Dr Dibyendu's one billed consult of 19 August is audible and readable:
the greeting, the history, the SLE, the medication and dose. The consultation that Pulse records as a
single row is now a conversation. That is the thing this slice existed to prove.

## 2. The defect

Two runs of the **same window on the same clip** — identical `r2_key`, identical 1,497,278 bytes.

| | run 1 | run 2 |
|---|---|---|
| segments returned | 162 | 165 |
| rows written | 71 | 6 |
| already existed | 0 | 71 |
| dropped | 0 | 0 |
| stopped early | time_budget | time_budget |

Run 2 wrote six new rows for a window it had already transcribed. Those are not new speech. They are
the same speech with different segment boundaries, so they are different `source_ref` values, so the
key treats them as different turns.

The text diverges too, and it diverges in one specific place: where Whisper falls into a repetition
loop. Run 1 produced one loop, run 2 produced a different one. Everything outside those loops matched.

This is your own gotcha, arriving from a direction none of us checked. `ON CONFLICT DO NOTHING` is
idempotency for a deterministic writer and accumulation for a non-deterministic one. We proved the key
against a deterministic writer, on clean single-chunk audio, and it held at 35 of 35. The messy audio
is where it fails, and messy audio is the whole point of the corpus.

## 3. The second defect

`stopped_early: "time_budget"` appears in the response and nowhere in the database. A reader of the
scratch day cannot tell a complete window from a truncated one. The Cardiology window holds 77 rows
against a true segment count of 162 or 165. Nothing in those rows says so.

Together the two defects mean the current writer cannot produce a trustworthy window: it stops
part-way, and re-running to finish it adds divergent rows rather than the missing ones.

## 4. State of the corpus, marked

**Do not analyse `rd_scratch_bh6jtq4t_20260819`'s turn rows.** 77 `stt_turn` rows, incomplete and
drawn from two disagreeing transcriptions. Left in place deliberately, as evidence of the defect, and
marked here so nobody reads them as a picture of the consult. They should be deleted before any fixed
writer runs, per your delete-before-re-running rule.

`rd_scratch_qyzghzaf_20260819` holds 35 `stt_turn` rows from one clean 2-minute window at 04:02Z.
That window is complete and stable. It is not the hole window you asked for. I did not run the hole
window, because each attempt would add divergent rows.

The fuse does not read `stt_turn`, and no visit depends on these rows, so nothing downstream is wrong
yet.

## 5. What we recommend

**Make the window the unit of write, not the turn.** Delete the existing turns for a
`(session_id, window)` pair, then insert the new set in one transaction. A re-run then replaces rather
than accumulates. This is the honest shape for a non-deterministic producer: the window is what we
asked for, the segmentation is an opinion about it, and an opinion should be replaced rather than
merged.

**Pin the decoder.** Ask whisper.cpp for greedy decoding, beam size 1, temperature 0 and a fixed seed.
This should reduce the churn at source and may suppress the repetition loops, which are the only place
the two runs disagreed. It will not make the writer deterministic on its own, which is why it is the
second recommendation and not the first.

**Make truncation visible.** Either raise the budget and write the whole window in one transaction, or
record on the day that a window is partial. A silent partial is worse than a refusal.

## 6. What we need from you

1. **Rule on the key.** Window-as-unit with delete-then-insert, or something else. Ruling 2 stands or
   changes on this.
2. **Rule on truncation.** Refuse a window we cannot finish, or record it as partial.
3. **Rule on cleanup.** We assume the 77 Cardiology rows get deleted before the fixed writer runs.
   Confirm.
4. **Then the hole window.** OPD 7 inside `04:36:59Z` to `10:39:35Z` is still unrun, and it is the
   window that decides whether six hours of tape hold speech or an empty room.

Everything else in rev 3 stands. The blocklist is live, the scratch guard held on every write, and the
tool defaults dry.
