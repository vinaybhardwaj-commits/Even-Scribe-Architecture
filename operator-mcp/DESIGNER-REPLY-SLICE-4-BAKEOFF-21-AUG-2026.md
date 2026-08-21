# Designer reply — slice 4 as a three-arm bake-off
21 Aug 2026. Scribe Designer. For the orchestrator.

Read `BUILDER-PROPOSAL-SLICE-4-BAKEOFF-21-AUG-2026.md`. You asked to change §11.4 before building it. Confirmed from this chat: live OPD 7 still `cue_95z4avnv`; live Cardiology still three kiosk marks; scratch Cardiology now has the three marks at `04:41Z`, `06:08Z`, `09:34Z`. Precondition met.

## Call

**Yes to the bake-off. Yes to 0048, with one key rewritten. I adjudicate on the day report. The model case is real, and it is not this corpus.**

§11.4 becomes: fuse on scratch as three arms (rules / hybrid / flash) against the same 19 Aug scratch days. Designer picks the winner from the slice 5 report. No arm writes the live `room_day`.

§7 does **not** become a state machine. Priors stay weights for the product fuse. This slice's input is only marks + warehouse clocks. On that input, interval arithmetic is the honest first arm, and a model that cannot re-run to the same answer cannot be accepted against §10 as written.

## Answers

**1. Three-arm shape — accept.** Arm A rules, arm B hybrid (A's visits, Gemini only on A's explicit ambiguous, never mints identity), arm C Gemini 3.7 Flash as §11.4 was written. Same scratch room-days. Same ten §10 rows. Disagreement is the interesting output.

**2. Migration 0048 — accept the column, reject the unique key as proposed.**

Accept:

- `visit.arm` nullable, `'rules' | 'hybrid' | 'flash'`
- arm filter on `SQL_VISITS_FOR_DAY` and on `active_visit_id`
- default arm on read so `GET /state` / `scribe_get_state` mean one picture
- unify the duplicated `SQL_VISITS_FOR_DAY` (`lib/brain/state.ts` and `brain/src/state.ts`) in the same change; do not widen the drift

**Reject** a partial unique on `(arm, room_day_id, individual_uid)`. That key forbids §10.3 (second `pstart` + new `calendar_uid` is a second visit for the same person on the same day) and cannot hold mark-only visits whose `individual_uid` is null.

Idempotency is per arm + the **opening evidence**:

- opened by `pstart` → `(arm, pstart source_ref)`
- opened by mark with no `pstart` → `(arm, mark cue id)`
- never unique on the person

Default arm on read, until I pick a winner, is **`rules`**. Not flash. Not last-`updated_at`. Not lexicographic id.

`speaker_cluster.visit_id` stays single. On this corpus voice is not a binder. Do not attach the same cluster to three visits. Skip cluster attach until a winner, or attach only to the default arm.

0048 is a bake-off scaffold, not a product field that lives on every future visit. After I pick, the other arms stay filtered out of default read. Do not write live.

**3. If the arms disagree, I adjudicate on the day report.** Do not encode a winner rule now. An arm that never returns low confidence on a six-hour evidence gap loses that row, however plausible. Determinism is reported. Cost is reported and not decisive.

**4. Cases the rules cannot express** — these are why the product fuse is still a model. None of them are in this corpus.

- Return from diagnostics with no PQM / no second `pstart`: same person, minutes later, only voice or content says so (good-enough bar, 18 Aug).
- Two open visits in one room when clocks do not split them (A at dx, B sits; mark without warehouse).
- Binding identity from STT when diarization is messy (LLM role assignment, locked 18 Aug). Speech turns are not in slices 1–3.
- Operator pin the fuse may reject. A rule that always adopts, or never does, is the second fuse we forbade.
- Patient-day thread across rooms / doctors. Visit key is `individual_uid` + IST date. A per-`room_day` state machine mints two visits. Slice 6, and this corpus's patient sets did not overlap.

On 4 marks + 43 warehouse events, if arm A cannot be silent where the evidence is silent, that is a bug in A, not an argument for C.

## Hard locks for B and C

The CDMSS silent-fallback story is in scope. If Gemini errors and `routedChat` returns Ollama, **do not write visits**. Arms B and C fail closed and report the provider they actually got. A fuse that is secretly `qwen2.5:14b` cannot be scored as Flash.

Gemini 3.7 Flash is the Flash we asked for. Do not pull Pro.

## Next

Build 0048 as rewritten, then the three arms on scratch only. Live OPD 7 cue count stays 1. Live Cardiology stays three kiosk marks.

Slice 5 (marks vs warehouse vs visits vs tape) is the scoreboard, and it must name the arm. Send that report before any winner is defaulted, and before any pass that writes the live `room_day`.
