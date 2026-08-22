# Slice A — the two-window report

22 August 2026. From the orchestrator, to the Scribe Designer.
Prod `c93078f`, migrations through `0053`.

Both windows are written and both report `complete: true`. The window-as-unit writer works. One
finding changes what OPD 7's six-hour hole means, and one design guarantee is not implemented.

No live `room_day` was written. No visit was invented. No Pulse write, no Slack, no Gerrit. No
transcript appears in this document.

---

## 1. The two windows

**Cardiology `bs_vqummntv`, 06:52:00Z–06:58:00Z** — the window around Dr Dibyendu's one billed consult.

```
deleted 77 · written 134 · already_existed 0 · dropped 0 · complete true
```

134 is 133 turns plus one `stt_window` marker. The 77 rows from the two disagreeing runs were swept by
the writer's own delete, exactly as you said they would be. This is the replace working on its first
real exercise, and it took 25 seconds where the row-at-a-time path took over 70 to write half as many.

**OPD 7 `bs_xvntaugh`, 06:00:00Z–06:06:00Z** — inside the hole, at 11:30 IST, prime clinic time.

```
deleted 0 · written 145 · already_existed 0 · dropped 0 · complete true
```

## 2. The hole is not an empty room

144 turns. **Zero silences.** Continuous speech across the whole six minutes.

It is a dermatology consultation carrying a complete prescription: a systemic drug with strength,
dosing days, timing relative to food and a stated duration; a topical with frequency and site; a lab
check; a review interval; and safety counselling covering immunity, what symptoms should stop the
drug, and what to do about a missed dose.

The warehouse holds **no event of any kind** across this window. Not a late clock, not a partial
record. Nothing.

So OPD 7's `04:36:59Z → 10:39:35Z` gap is not the clinic being quiet. It is care being delivered and
not recorded. One six-minute probe of a six-hour hole found a full consultation, which makes the
remaining five hours and fifty-four minutes the question rather than the answer.

## 3. What broke, and what it taught

The first attempt at the hole window failed. Every count zero, `stopped_early: "brain_unavailable"`,
and no rows at all.

The cause was not code. **The brain's database role had never been granted DELETE on `cue`.** Every
write in this system's history had been INSERT-only; K3's window-as-unit replace is the first code
ever to delete a cue. The role authenticates and reads fine, so every health check passed. The
evidence was one line in the Vercel runtime log, `permission denied for table cue`, which the HTTP
response discarded.

The fix is migration `0053`, and the reason it had to be a migration is worth recording: Vercel will
not export the owner database URL, so the CLI has no owner connection. The migration runner **is** the
owner. A migration is therefore not merely the tidy place for grants, it is the only owner-privileged
path that exists.

DELETE is now granted on `cue`, `room_day`, `visit` and `speaker_cluster`. `room` stays SELECT-only,
because the brain has never written a room.

**Your refuse-partial rule worked on its first real exercise.** Nothing was committed. No 71 of 144.

## 4. The guarantee that is not implemented

Your §2 says a `complete: false` marker must land so a reader can tell "we asked and failed" from "we
never asked."

It did not land. We verified: zero `stt_window` rows on the day after the failure.

The reason is structural. The fallback marker write reuses the identical machinery as the primary
write, including the same `replace_window`, and therefore the same DELETE. So it shares every failure
mode with the write it is supposed to report on. When the cause of failure is structural rather than
size-dependent, the fallback is guaranteed to fail too, and the graph is left silent.

That is what happened. Both posts died on the same missing grant.

**Rule needed.** Our recommendation: the marker should be written through a path that does not depend
on the delete, so a failure of the replace can still be recorded. That means either a marker write
with no `replace_window`, or an explicit second door for it. Either way the marker must not inherit
the verbs the turn write needs.

## 5. Two smaller findings

**The decoder pin improved accuracy, not only determinism.** The Cardiology window is the first to run
under greedy decoding with `best_of 1` and temperature 0. The earlier sampled runs rendered the
patient's diagnosis as a non-word, and collapsed into repetition loops. The pinned run reads the
diagnosis correctly and drops the loops. Temperature sampling was inventing a wrong clinical term.
This raises the value of the pin above what we claimed when we recommended it.

**Instability is a property of the audio, not of Whisper.** The OPD 7 hole window produced the same
144 segments and the same text before the pin and after it. Only the Cardiology window, with
overlapping speakers and cross-talk, was unstable. That argues for keeping the window as the write
unit permanently rather than relaxing it once the pin is in.

**`dropped` still carries two meanings.** On the failure path it is assigned the turn count verbatim,
so a transport or permission failure reports as 144 drops. Your contract reserves a drop for a named
within-write conflict, which is a bug. A reader following the contract would have read that number as
144 key collisions. Worth separating.

## 6. Where it stands

- Cardiology `rd_scratch_bh6jtq4t_20260819`: one complete window, 133 turns, marker `complete: true`.
  The 77 bad rows are gone.
- OPD 7 `rd_scratch_qyzghzaf_20260819`: the hole window, 144 turns, marker `complete: true`. Plus the
  35 stable rows at 04:02Z from the earlier proof window, untouched.
- Live OPD 7 still holds one cue. Live Cardiology still holds three kiosk marks and zero visits.

Awaiting your ruling on the marker before we walk any more of the day.
