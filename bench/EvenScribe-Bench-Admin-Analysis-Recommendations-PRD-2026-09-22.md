# Even Scribe — Bench Admin (`/admin/bench`)
## Analysis, Recommendations & PRD
**Date:** 22 Sep 2026 IST  
**Author:** EvenScribe Designer  
**Live surface:** https://www.evenscribe.app/admin/bench  
**Code map source:** artifacts/bench-admin-code-analysis.md (PR #4 analysis-only on Even-Transcription-Assistant)

---

### § Live UI audit (22 Sep 2026 IST)

**Walk conditions.** Admin `vinay.bhardwaj@even.in`. Mid-afternoon clinic load on `https://www.evenscribe.app/admin/bench`. **No production mutations confirmed.** One paid-run confirm opened accidentally, then abandoned (dismissed). Screenshots copied to `screens/` (appendix §4.6).

#### IA / nav (sidebar)

| Group | Items observed |
|---|---|
| **Operate** | Dashboard, Clinicians, Encounters |
| **Observe** | LLM traces, Sends, Diarization, STT Lab, **Bench** (active), Rooms |
| **Configure** | Admins, System map, Settings |

**Bench hierarchy (single page, top→bottom):** live monitor → attention → room cards → day summary → Install/fleet → session table (keyed to selected card) → rooms CRUD. **Separate Rooms hierarchy** for aggregate / day / slots (technical STT / voice-match surfaces). Session drill-down: `/admin/bench/[id]`.

#### Live monitor — what is on the page

- **Header:** screen awake, date (Tue 22 Sep 2026), refresh / last-refresh clock.
- **Dominant control:** large red **Stop all processing** (visually primary; High blast-radius).
- **Attention list:** Dietary — no kiosk + no audio arriving (act-now); many rooms — transcript backlog with **audio safe** copy.
- **Room cards (two-column):** Tape / Transcript / Visits toggles; waiting audio; **Run waiting (paid)**; mic freshness; Start / Pause / Resume / Stop.
- **Observed mid-afternoon states:** Dietary — “Says recording, silent ~15m” + **Close abandoned session**; other rooms recording with piece counts + waiting queues; Home Office finished / device issues in fleet.
- **Day totals (observed):** ~**59h45m** recorded / ~**11h30m** turned into words.
- **Install / fleet table:** Room, Machine, App, Last seen, Microphone, Tape — plus recording-input / volume cues; **Copy install**, **Retire**, **Withdraw**. Release **0.1.24**. **HO device missing** shown in fleet (not promoted into attention).
- **Sessions table:** filtered to selected room card.
- **Rooms CRUD** table on the same page.

#### Session detail & Rooms

- **`/admin/bench/[id]`:** downloads, chunk timeline, gaps, empty consult marks.
- **Rooms list / day:** technical STT / voice-match surfaces (separate Observe → Rooms tree).

#### Strengths (compressed from live walk)

1. Attention strip already ranks Dietary desync and backlog-with-audio-safe ahead of a flat card dump.
2. Room cards carry the mid-day control surface (lanes + transport + paid run) in one scan.
3. Screen-awake + refresh age make poll lag partially honest.
4. Day totals separate **recorded** vs **turned into words** — useful clinic wrap signal.
5. **Install/fleet section exists on the page** (native Mac/app presence, copy-install, retire/withdraw) — richer than the code map’s “not a native fleet board” framing implied.
6. Careful confirms still present on stop / visits-ON / paid-run (paid-run confirm was opened and abandoned safely).
7. Abandoned-session close is surfaced on Dietary when session is open but kiosk is not recording it.

#### Problems (High / Med / Low) — live walk only

| Sev | Problem |
|---|---|
| **High** | **Stop all processing** is visually dominant chrome on a monitoring board — easy mis-tap under clinic pressure. |
| **High** | **Write controls mixed into monitoring** (per-card Start/Pause/Resume/Stop, lane toggles, paid Run, Stop-all) without a clear observe-vs-act zoning. |
| **High** | **`DEVICE_MISSING` / HO offline buried in fleet**, not elevated into the attention strip beside Dietary. |
| **Med** | **Waiting-unit confusion** — “pieces waiting” vs time (“15m”, “turned into words 15m”) compete; paid-run cost implication easy to miss mid-scan. |
| **Med** | Dietary desync is correctly in attention, but the card still reads “recording” while silent — twin truths without a single desync chip. |
| **Med** | Bench vs Rooms dual hierarchy — operator must leave Bench for aggregate/day/slot STT surfaces. |
| **Low** | Session table peer position still competes with live cards for scroll attention. |
| **Low** | Empty consult marks on session detail add noise when unused. |

#### Missing affordances (live)

- Attention promotion for **DEVICE_MISSING / host-offline / silence-while-recording** (HO never reached the attention list).
- Explicit **waiting unit** (pieces vs minutes) and paid-batch cost preview before confirm.
- Demoted / separated **Stop-all** (danger zone, not header peer).
- Per-command **ack / timeout** outcome on the card (still write-into-the-dark from the walk — no delivery UX observed).
- Deep link **`?room=`** (could not share a room from URL during walk).

#### Confusing labels (live)

- **Stop all processing** vs recording — copy must stay unmistakable that tape continues; visual weight currently says “kill everything.”
- **Run waiting (paid)** / “pieces of audio waiting” — unit and cost not scannable at card glance.
- **Says recording, silent Nm** vs attention “no audio arriving” — same failure, two phrasings.
- Fleet **device missing** language disconnected from attention vocabulary.

#### Live vs code-map corrections (do not rewrite code facts)

| Code-map framing | Live UI confirmed |
|---|---|
| Board “not a native Mac fleet board” / missing fleet scope clarity | **Fleet/install section EXISTS** on `/admin/bench` (understated in code map). Scope decision remains open; UI already shows install/retire/withdraw + app version. |
| `DEVICE_MISSING` not a Bench badge | Present as **fleet row state**, still **not** an attention chip. |
| Attention ranking exists | Confirmed; Dietary + backlog/audio-safe live. Silence / DEVICE_MISSING promotion still incomplete. |
| Stop-all + processing switches present | Confirmed; Stop-all is **High visual risk**. |
| Write-only bus / ack opacity | Confirmed from walk — no per-card delivery outcome observed. |


---

## 0. Executive summary (½ page)

**What it is today.** `/admin/bench` is an admin-JWT supervisory board for room-day tape and selected-room session history. It combines a large live monitor (`BenchRoomsLive`, ~1589 lines) with a session table (`BenchClient`, ~604 lines). Session drill-down lives at `/admin/bench/[id]`. There are no WebSockets: the page polls listeners (~3s), rooms-live (~20s), session/rooms list (~60s), and ticks age every 1s. Writes go onto a command bus (start / pause / resume / stop, `close_orphan`) plus processing switches, stop-all, run-waiting audio (batch 4), and room CRUD. The UI does not wait for kiosk acknowledgment (MCP waits ~8s). Pause/stop are not listener-gated on admin (MCP is). There is no `override_pause` from the UI. Drain/windows APIs exist server-side but are unused by this page.

**Biggest gaps for a mid-day clinic operator (Vinay / Bug Bot style).** Operators watching ~10 rooms need failure classes that already hurt clinic ops (Sep 21–22 research): silence-while-recording, `ENCODER_STALLED`, `DEVICE_MISSING` mid-record, host/cloud desync (`session_open` false vs brain Recording), kiosk offline NEED-WAKE, `tape_without_cues`, `physical_fallback` after coreaudiod, and Cardio TONOR digital silence with pieces still advancing. Bug Bot and MCP often surface these sooner than Bench UI. Soft restart does not fix USB/coreaudiod; operators need a host RCA path. The current board is a write-only bus with ack opacity, missing status chips for the real failure vocabulary, twin clocks/lists that fight attention, a giant component, a 200-cap session list, and no `?room=` deep link. **Live UI (22 Sep IST) confirmed:** fleet/install section **exists** on the page (code map understated native fleet UI); **Stop all processing** is visually dominant (**High** risk); write controls are mixed into monitoring; **`DEVICE_MISSING` (HO) is buried in fleet, not attention**; Dietary desync is visible in attention; waiting units (pieces vs time / paid run) confuse mid-scan.

**Redesign north star (3 bullets).**

1. **Fleet board first** — room cards as the primary product; session table becomes drill-down / archive, not a peer surface competing for attention.
2. **Honest command outcomes** — every write shows queued → ack / timeout / conflict; align admin write semantics with MCP or label “queued into the dark.”
3. **Ops-complete status dictionary** — expand beyond today’s seven `roomState()` values to chips operators already chase: digital silence (peak/zr), `DEVICE_MISSING`, host/cloud desync, `tape_without_cues`, encoder stalled — without dumping drain jargon onto the board.

---

## 1. Analysis — what it is today

### 1.1 Information architecture

| Surface | Role | Scale (code map) |
|---|---|---|
| `/admin/bench` | Live monitor + session table | `BenchRoomsLive` ~1589 lines; `BenchClient` ~604 lines |
| `/admin/bench/[id]` | Session detail + consult marks | Separate route |
| Auth | Admin JWT | Assumed gate for all writes/reads |

**Mental model (confirmed):** room-day / session **supervisory board** — tape for today plus selected-room session history. It is **not** a clinician roster and **not** an encounter brain. **Live correction (22 Sep):** an **Install/fleet** section *does* appear on `/admin/bench` (app version, last seen, mic, copy-install, retire/withdraw). Treat “not a native fleet board” as a **product-scope** statement from the code map, not as “fleet UI absent.” Whether Bench *owns* host RCA remains an open Vinay decision (§3.14).

### 1.2 Data freshness (polling only)

| Poll | Interval | Purpose |
|---|---|---|
| Listeners | ~3s | Mic / listener health inputs |
| Rooms-live | ~20s | Live room facts for cards |
| Session / rooms list | ~60s | Archive / table refresh |
| Age tick | 1s | Relative age UI only |

No WebSockets. Freshness UX is entirely poll-driven; stale windows between polls are invisible unless age ticks make lag obvious.

### 1.3 Writes available from the page

| Action | Notes from code map |
|---|---|
| Command bus: start / pause / resume / stop | UI never waits for kiosk ack (MCP waits ~8s) |
| `close_orphan` | On bus |
| Processing switches + stop-all | Present |
| Run-waiting audio | Batch of 4 |
| Room CRUD | Present |
| `override_pause` | **Not** from UI |
| Drain / windows APIs | **Unused** by this page |

**Write-semantics gap:** Pause/stop are **not** listener-gated on admin; MCP **is** listener-gated. Operators can fire commands that MCP would refuse or sequence differently. Ack is opaque — the board is effectively write-only into the dark.

### 1.4 Status model today

- `roomState()` exposes **7 states** (exact enum labels: see flag dictionary excerpt / code analysis; do not invent names here beyond those confirmed below).
- **Stalled** = recording + no piece from either mic for **10+ minutes** (rendered red).
- **`DEVICE_MISSING`** is a mic-health / kiosk remount reason — **not** a Bench attention badge today. **Live:** HO shows device-missing inside **fleet**, still not promoted into the attention strip.
- **`tape_without_cues`**, **`paused_disagrees`**, and **command queue** are **not on screen**.
- **Doctor clock** rarely draws.

### 1.5 Strengths

1. **Shared room-facts with MCP** — single contract potential; operators and agents can reason over the same facts if the UI surfaces them.
2. **Attention ranking** — some prioritization already exists so the board is not a flat dump. **Live:** Dietary desync + backlog/audio-safe rows confirmed mid-afternoon.
3. **IST day cards** — clinic-local day semantics match operator mental model (Asia/Calcutta).
4. **Careful confirms** on stop and visits-ON — high-blast-radius actions are not one-click accidents. **Live:** paid-run confirm also present (opened accidentally, abandoned; no mutation).
5. **Admin JWT boundary** — clear privileged surface; not mixed into clinician UX.
6. **Install/fleet on-page (live)** — copy-install / retire / withdraw / 0.1.24 / mic health exist on Bench itself; code map understated this native-facing block.

### 1.6 Weaknesses

| Weakness | Why it hurts mid-day ops |
|---|---|
| Giant `BenchRoomsLive` (~1589 lines) | Hard to evolve cards, status chips, and command UX independently |
| Two clocks / two lists | Operator attention splits between live monitor and session table |
| Write-only command bus | No per-card outcome; silent failures; MCP/Bug Bot ahead of UI |
| Fleet UI present but scope unclear (live) | Install/fleet exists; `DEVICE_MISSING` still buried there; host RCA ownership undecided |
| Missing / under-ranked badges: silence, `DEVICE_MISSING`, desync, `tape_without_cues`, encoder stalled | Real Sep 21–22 failures invisible or under-ranked; **live:** HO missing not in attention; Dietary desync *is* in attention |
| Ack opacity | “Did stop land?” unknown without leaving Bench |
| Session list 200-cap | Evening wrap / archive search can miss older sessions without warning clarity (exact overflow UX: unknown from code map) |
| **Stop all processing** visually dominant (live) | High mis-tap risk on a mid-day monitoring board |
| Write controls mixed into monitoring (live) | Observe and act share one scroll; no danger zoning |
| Waiting-unit confusion (live) | Pieces vs minutes vs paid-run cost not scannable |
| No deep link `?room=` | Cannot share “look at Cardio-2” in Slack / MCP handoff |

### 1.7 Operator journey gaps (clinic research Sep 21–22)

Treat as user research from live clinic ops (designer memory), not as instrumented telemetry.

| Failure pattern | Operator need | Bench UI today (from code map + research) |
|---|---|---|
| Silence-while-recording / digital silence (TONOR; pieces still advancing) | Peak/zr chip + severity | Not a first-class badge |
| `ENCODER_STALLED` | Distinct from “stalled pieces” if different | Stalled = 10+ min no piece; encoder-specific: unknown as separate chip |
| `DEVICE_MISSING` mid-record | Remount / host path, not soft restart theater | Mic-health reason only; not Bench badge |
| Host/cloud desync (`session_open` false vs brain Recording) | Explicit desync chip | Not on screen |
| Kiosk offline NEED-WAKE | Wake / host RCA path | Soft restart does not fix USB/coreaudiod |
| `tape_without_cues` | Cue health visible | Not on screen |
| `physical_fallback` after coreaudiod | Host RCA, not kiosk-only restart | Path not on board |
| Bug Bot / MCP ahead of UI | UI should not lag agent vocabulary | Write semantics + status dictionary diverge |

**Journey gap summary:** Mid-day operator scans ~10 rooms, spots red, fires pause/stop/restart, then waits without ack. When soft restart fails (USB/coreaudiod), Bench gives no host RCA ladder. Evening wrap leans on session table / day cards but hits 200-cap and lacks deep links for async handoff. Remote MCP agent already has stricter gating and ack wait — Bench admin undercuts that contract.

### 1.8 What this page is *not*

- Not clinician roster / scheduling.
- Not encounter brain / note quality surface.
- Not a full native Room Recorder / host-RCA board (Install/fleet UI exists on-page; ownership of host remediation still open — see recommendations).

---

## 2. Recommendations (prioritized)

### P0 — Ship before more feature paint

| # | Recommendation | Rationale (clinic ops) |
|---|---|---|
| P0.1 | **Fleet board as primary product; session table = drill-down / archive** | Mid-day eyes need room cards, not a peer table. Reduce dual-clock competition. |
| P0.2 | **Surface bus outcomes per card** (queued → ack / timeout / conflict) | Stops “write into the dark.” Matches how Bug Bot/MCP already reason. |
| P0.3 | **Align write semantics with MCP** *or* honest label **“queued into the dark”** | Pause/stop listener-gating mismatch causes false confidence. Pick one contract. |
| P0.4 | **Status dictionary expansion** — chips for: digital silence (peak/zr), `DEVICE_MISSING`, host/cloud desync, `tape_without_cues`, encoder stalled | These are the Sep 21–22 failure classes operators already chase. |
| P0.5 | **Deep link `?room=`** | Slack / MCP / Bug Bot handoffs to a specific room without narrative scavenger hunts. |
| P0.6 | **Demote / separate Stop-all** from live-monitor chrome (danger zone + strong confirm; never header-peer red) | Live walk: Stop-all is visually dominant High risk on a monitoring board. |
| P0.7 | **Promote `DEVICE_MISSING` / host-offline / silence-while-recording into attention** (not only fleet rows) | Live: HO device-missing buried in fleet; Dietary silence already in attention — make the vocabulary consistent. |
| P0.8 | **Clarify waiting units** (pieces vs minutes) and paid-run cost at card glance | Live: waiting-unit confusion mid-scan; paid batch easy to misread. |

### P1 — Structural & scope

| # | Recommendation | Rationale |
|---|---|---|
| P1.1 | **Split `BenchRoomsLive`** into card grid, status/attention, command panel, polling shell | 1589-line monolith blocks safe iteration on P0 chips/outcomes. |
| P1.2 | **Decide kiosk-only vs native fleet scope** (product decision with Vinay) | Soft restart ≠ host RCA. **Live:** Install/fleet UI already on Bench — decide whether that block owns host RCA or is install-only. |
| P1.3 | **Keep room-facts as the single contract** with MCP | Avoid a second parallel status inventing Drift between agent and human. |

### P2 — Hygiene & home for design work

| # | Recommendation | Rationale |
|---|---|---|
| P2.1 | **Don’t put drain jargon on the board** | Drain/windows APIs unused by page; operators don’t need internal pipeline terms mid-clinic. |
| P2.2 | **Move analysis/PRD home to Even-Scribe-Architecture** | Designer never ships product PRs. PR #4 on Even-Transcription-Assistant should **stay analysis-only** or be **closed after copying** artifacts into Architecture. |

### Recommendation principles (non-negotiables)

1. Design-only in this doc — no code patches as “implementation steps.”
2. Accurate to verified code map; unknowns stay **unknown**.
3. Severity model may map lightly to Bug Bot S-classes without locking the product to Bug Bot.

---

## 3. PRD — Bench Operator Fleet Board v1

### 3.1 Problem

Clinic operators supervising ~10 Even Scribe rooms mid-day cannot reliably see the failure classes that lose tape (silence-while-recording, device missing, encoder stalled, host/cloud desync, cue starvation, offline NEED-WAKE). The current `/admin/bench` board polls shared room-facts but under-surfaces them, accepts commands without ack UX, and diverges from MCP write semantics. Soft restart theater does not fix USB/coreaudiod; the board does not guide host RCA. Session history competes with live attention and lacks deep links for handoff.

### 3.2 Goals

1. Make **room fleet** the default mid-day surface with scannable severity.
2. Make every command **outcome-visible** on the room card.
3. Expand status chips to the **ops vocabulary** operators already use (aligned with room-facts).
4. Provide **`?room=`** deep links for human and agent handoff.
5. Clarify **kiosk vs native/host** remediation paths without pretending soft restart fixes hardware.

### 3.3 Non-goals

- Replacing encounter brain, note QA, or Sentiment surfaces.
- Shipping native Room Recorder Mac UI inside Bench v1 (may be v1.1 if Vinay chooses native fleet scope).
- Exposing drain/windows pipeline controls on the operator board.
- WebSocket rewrite as a v1 requirement (polling freshness UX is enough if honest).
- Locking severity taxonomy 1:1 to Bug Bot forever.
- Designer-authored product PRs on Even-Transcription-Assistant.

### 3.4 Personas

| Persona | Context | Primary need |
|---|---|---|
| **Clinic operator (mid-day)** | Watching ~10 rooms; interruptions; Slack pings | Scan → spot failure chip → command with confirm → see ack/timeout |
| **Evening wrap operator** | Closing day; orphans; missed sessions | Session archive, day cards (IST), close_orphan, run-waiting — without losing live context |
| **Remote MCP agent** | Same room-facts; stricter gating; ~8s ack wait | Contract parity with admin UI; deep links; no silent admin overrides that MCP would refuse |

### 3.5 Jobs to be done

1. When a room looks “recording” but audio is dead, I need to **see digital silence / stalled / DEVICE_MISSING** without opening MCP.
2. When I stop or pause a room, I need to **know if the kiosk accepted** or if I queued into the dark.
3. When Bug Bot / a teammate says “Cardio-2,” I need a **link that opens that room’s drawer**.
4. When soft restart fails after coreaudiod, I need a **host RCA path**, not another pretend kiosk button.
5. When evening wrap starts, I need **archive + orphans** without losing the fleet mental model.

### 3.6 Information architecture

```
/admin/bench                      Fleet Operator Board (default)
├── Fleet grid                    Room cards + attention sort
├── Room drawer                   Detail, chips, command outcomes, recent tape facts
├── Command drawer / panel        Confirms + bus outcome log for selection
└── Session archive               Drill-down table / IST day cards (secondary)

/admin/bench/[id]                 Session detail + consult marks (keep)
/admin/bench?room=<id>            Deep link → opens room drawer for <id>
```

**IA rule:** Fleet grid is primary. Session archive is secondary. Do not present two equal “home” lists.

### 3.7 Room card anatomy

**Must-show fields**

| Field | Notes |
|---|---|
| Room name / id | Stable label for Slack handoff |
| Live status chip | From expanded dictionary (below) |
| Recording / session open indicator | Explicit; supports desync detection |
| Piece freshness / age | Age tick already exists — make lag obvious |
| Mic / listener health summary | Include DEVICE_MISSING when present in facts |
| Last command + outcome | queued / ack / timeout / conflict / unknown |
| Attention rank affordance | Why this card is elevated |

**Status chips (v1 dictionary — map to room-facts; do not invent APIs)**

| Chip | Intent | Source expectation |
|---|---|---|
| Existing `roomState()` (7) | Keep baseline | Current `roomState()` |
| Stalled (pieces) | Recording + no piece either mic ≥10 min (red) | Already defined |
| Digital silence (peak/zr) | Pieces advancing but energy dead | Room-facts / mic health if available; else unknown until fact exists |
| `DEVICE_MISSING` | Remount / host path — **promote to badge** | Today mic-health only |
| Host/cloud desync | e.g. `session_open` false vs brain Recording | Compare facts already shared with MCP |
| `tape_without_cues` | Cue starvation | Fact exists for agents; surface on card |
| Encoder stalled | Distinct when facts distinguish from piece-stall | If not distinguishable in facts: label unknown / merge carefully |
| NEED-WAKE / offline | Kiosk unreachable | Listener / kiosk health |

**Must not show on card:** drain jargon, windows pipeline internals, clinician PII beyond operational need.

### 3.8 Attention / severity model

Light map to Bug Bot S-classes **without locking**:

| Severity band | Operator meaning | Examples (ops) | Suggested UI weight |
|---|---|---|---|
| S0 / Critical | Tape at risk now | Digital silence while “recording”; DEVICE_MISSING mid-record; encoder stalled; offline NEED-WAKE while expected live | Top of grid; red chip; optional pulse |
| S1 / High | Desync or cue failure | Host/cloud desync; `tape_without_cues`; paused_disagrees (when surfaced) | High sort; amber/red |
| S2 / Watch | Aging / soft anomalies | Piece age climbing; doctor clock missing (rarely draws today) | Elevated but not alarming |
| S3 / OK | Healthy recording / idle expected | Steady pieces + cues; idle outside clinic hours | Default sort |

Exact Bug Bot class names are **not** a product dependency; shared room-facts are.

### 3.9 Commands & confirms

| Command | Confirm? | Outcome UX | Semantic note |
|---|---|---|---|
| Start | Low / contextual | Per-card queued→ack/timeout | Align gating with MCP or label dark-queue |
| Pause | Confirm if mid-tape risk | Same | Admin today not listener-gated; MCP is — **resolve** |
| Resume | Low | Same | |
| Stop | **Strong confirm** (keep today’s care) | Same | High blast radius |
| `close_orphan` | Confirm | Same | Evening wrap |
| Processing switches / stop-all | Strong confirm | Global outcome banner + per-room where possible | |
| Run-waiting audio (batch 4) | Confirm batch size | Progress of batch | Keep batch semantics visible |
| Room CRUD | Confirm destructive | Standard | Not mid-day primary |
| `override_pause` | Out of UI today | If ever added: MCP-parity + audit | Unknown product need — open question |

**Principle:** If admin cannot wait for ack like MCP (~8s), the UI must say **queued into the dark** until timeout/poll proves otherwise.

### 3.10 Polling / freshness UX

| Signal | Interval (today) | UX requirement |
|---|---|---|
| Listeners | ~3s | Show last-success age on health chips |
| Rooms-live | ~20s | Card “facts as of …” when > interval + skew |
| Session/rooms list | ~60s | Archive stale indicator |
| Age tick | 1s | Relative ages only — do not fake live WS |

Degraded poll (errors / 401): banner with auth vs network distinction (see empty states).

### 3.11 Empty / degraded / auth states

| State | Behavior |
|---|---|
| No rooms | Empty fleet with CTA to room CRUD / setup docs (pointer only) |
| No sessions today | Archive empty; fleet still primary |
| Partial facts | Chip = unknown; never invent healthy |
| Poll failure | Non-blocking banner; last-known cards dimmed with timestamp |
| Admin JWT missing / expired | Hard gate; no silent read-only fake |
| Command timeout | Card outcome = timeout; suggest MCP/host path when DEVICE_MISSING / coreaudio class |

### 3.12 Success metrics

| Metric | Direction | Why |
|---|---|---|
| Time-to-notice for silence / DEVICE_MISSING / stalled | ↓ | Core mid-day job |
| Commands with visible terminal outcome within N seconds | ↑ | End write-only bus |
| Handoffs using `?room=` links | ↑ | Slack / MCP coordination |
| Soft-restart loops on USB/coreaudiod incidents | ↓ | Board should push host RCA, not theater |
| Operator reliance on Bug Bot solely for chips Bench should show | ↓ | UI catches up to agent vocabulary |
| Dual-list confusion reports / time-on-archive mid-day | ↓ | Fleet-primary IA working |

Instrumentation details: unknown; define with Vinay/builders.

### 3.13 Phased delivery

#### v0.1 — Quick wins on current page (no full redesign)

- Add missing chips where room-facts already exist (`DEVICE_MISSING`, `tape_without_cues`, desync if computable, silence if peak/zr present).
- Per-command outcome toast/row (queued / timeout) even without full ack protocol.
- Honest copy when admin write is not MCP-gated.
- `?room=` deep link → scroll/highlight card.
- Cap/overflow messaging for session list 200-cap.
- **Do not** add drain jargon.

#### v1 — Fleet Operator Board redesign

- Fleet grid primary; archive secondary.
- Room drawer + command drawer anatomy as specified.
- Split `BenchRoomsLive` into maintainable parts (builder task; designer specifies boundaries only).
- Severity/attention model live-sorted.
- Documented write-semantics parity decision implemented in UX.

#### v1.1 — Native fleet (optional)

- Only if Vinay decides Bench owns native Mac fleet / host RCA.
- Host path affordances for USB/coreaudiod / physical_fallback.
- Else: explicit “kiosk-only” scope + pointer to host runbook outside Bench.

### 3.14 Open questions for Vinay

1. **Scope:** Is Bench v1 **kiosk-only**, or does it own **native Mac fleet / host RCA**?
2. **Write contract:** Force **MCP parity** (listener-gate pause/stop + ack wait) or keep fire-and-poll with **“queued into the dark”** labeling?
3. Should **`override_pause`** ever appear on admin UI?
4. Is **200-cap** on sessions acceptable with clear overflow UX, or do we need server pagination?
5. Which silence signal is canonical for chips — peak, zr, both, or “unknown until fact lands”?
6. How tightly should severity labels mirror **Bug Bot S-classes** in operator-facing copy?
7. Confirm PR #4 on Even-Transcription-Assistant: **remain analysis-only** vs **close after copy** into Even-Scribe-Architecture.
8. Who owns evening-wrap SLAs for `close_orphan` and run-waiting batch visibility?

### 3.15 Acceptance criteria checklist

**v0.1**

- [ ] `DEVICE_MISSING` visible as Bench badge when present in room-facts
- [ ] `tape_without_cues` visible when present
- [ ] Host/cloud desync chip when facts disagree (or explicit unknown)
- [ ] Digital silence chip when peak/zr (or documented unknown)
- [ ] Encoder stalled distinguishable **or** documented merge with piece-stall
- [ ] Every bus command from UI shows queued → terminal outcome (ack/timeout/conflict/unknown)
- [ ] Copy discloses non-MCP gating where applicable
- [ ] `?room=` opens/highlights target room
- [ ] Session 200-cap communicates overflow/ truncation
- [ ] No drain/windows jargon on board
- [ ] Stop and visits-ON retains careful confirm

**v1**

- [ ] Fleet grid is default landing; archive is secondary navigation
- [ ] Room card shows must-show field set
- [ ] Room drawer + command drawer match IA
- [ ] Attention sort follows severity bands
- [ ] Polling freshness ages visible on degraded polls
- [ ] Empty / auth / degraded states implemented per table
- [ ] `BenchRoomsLive` split boundaries agreed with builders (design spec)
- [ ] Success metrics dashboard or log review plan agreed
- [ ] Analysis/PRD living in Even-Scribe-Architecture; product PR hygiene respected

**v1.1 (if in scope)**

- [ ] Native/host RCA path for USB/coreaudiod / physical_fallback
- [ ] Soft restart not presented as fix for those classes

---

## 4. Appendix

### 4.1 File map (primary files from code analysis)

> From `artifacts/bench-admin-code-analysis.md` (PR #4 analysis-only). Paths as named in that analysis; if a path is uncertain, marked unknown.

| Area | Primary artifact (code map) | Notes |
|---|---|---|
| Live monitor | `BenchRoomsLive` (~1589 lines) | Giant component; split candidate |
| Session table | `BenchClient` (~604 lines) | Peer today; should become archive |
| Session detail | `/admin/bench/[id]` route + consult marks | Keep |
| Room state helper | `roomState()` (7 states) | Baseline dictionary |
| Stalled rule | recording + no piece either mic ≥10 min | Red |
| Command bus clients | start/pause/resume/stop, `close_orphan` | UI no ack wait |
| Processing controls | switches + stop-all | Confirm carefully |
| Run-waiting | batch 4 | |
| Room CRUD | present | |
| Drain / windows | APIs exist; **unused by page** | Keep off board |
| Polling hooks | listeners 3s; rooms-live 20s; list 60s; age 1s | No WS |

Exact repo file paths beyond component names: refer to PR #4 artifact; do not invent.

### 4.2 Flag dictionary excerpt

| Flag / state | Bench admin today | Notes |
|---|---|---|
| `roomState()` ×7 | On screen | Baseline |
| Stalled (10+ min no piece) | Red | Defined |
| `DEVICE_MISSING` | **Not** a Bench badge | Mic-health / kiosk remount reason |
| `tape_without_cues` | **Not** on screen | Ops pain |
| `paused_disagrees` | **Not** on screen | |
| Command queue | **Not** on screen | |
| Doctor clock | Rarely draws | |
| `ENCODER_STALLED` | Ops research term | Distinct chip only if facts support |
| Digital silence / peak / zr | Ops research | TONOR case: pieces advance, energy dead |
| Host/cloud desync | Ops research | `session_open` vs brain Recording |
| NEED-WAKE / kiosk offline | Ops research | Soft restart ≠ USB/coreaudiod fix |
| `physical_fallback` | Ops research | Host RCA path |
| Drain / windows | Unused by page | Do not surface as operator chips |

### 4.3 Out of scope pointers

| Surface | Why out of Bench Operator Fleet Board v1 |
|---|---|---|
| **Room Recorder native UI** | Separate product surface; only enters via explicit v1.1 native fleet decision |
| **Encounter brain** | Clinical note / encounter reasoning — not supervisory tape board |
| **Sentiment** | Downstream quality / experience — not mid-day fleet ops |

### 4.4 Process note (designer hygiene)

- This deliverable is **design-only**.
- Living home for analysis/PRD: **Even-Scribe-Architecture**.
- PR #4 on **Even-Transcription-Assistant** should remain **analysis-only** or be **closed after copying** artifacts; designer does not ship product PRs.


### 4.6 Live UI screenshots (22 Sep 2026 IST)

Copied from `/workspace/screenshots/shot-call_*.png` into `/workspace/bench-audit/screens/` (15 files). No production mutations in the walk; one paid-run confirm opened then abandoned.

- `screens/shot-call_2kN6BEacUJEeBhWbgomS3qXyfc_0fd14e2a9863ec86.png`
- `screens/shot-call_D1UY4D8AB6vC03S7TCiz3mSvfc_0d75945961e2363e.png`
- `screens/shot-call_J2UTqKIWXiui3nXPNqhz9CmYfc_0d75945961e2363e.png`
- `screens/shot-call_JF8aL01BKV01YJ1P8ruqVkKffc_0d75945961e2363e.png`
- `screens/shot-call_JrZdzHljtc4zdUlspiHyunkufc_0d75945961e2363e.png`
- `screens/shot-call_KB1Rnq4ZiCLtUk89YCN13Ctkfc_0d75945961e2363e.png`
- `screens/shot-call_Lr6yan6NJgIG3lcC8MgTeNUlfc_0d75945961e2363e.png`
- `screens/shot-call_OccMJ2GtKxJh6v6xUrmX3ltBfc_0d75945961e2363e.png`
- `screens/shot-call_ebWRMh0IwVAO2ISdNjRV64vlfc_0d75945961e2363e.png`
- `screens/shot-call_mtkW2r3mrD8CqWDpU7yBl2Erfc_0d75945961e2363e.png`
- `screens/shot-call_oECEGOuJRnhaLrZ7u5sH2Kqrfc_0d75945961e2363e.png`
- `screens/shot-call_tJIWiU99TSk9EkAs6vwRHtT4fc_0d75945961e2363e.png`
- `screens/shot-call_uuVAt2PaKrxgyguJexPAagwWfc_0d75945961e2363e.png`
- `screens/shot-call_vCK2kn4btgEgBkOOrUyJ4lIFfc_0d75945961e2363e.png`
- `screens/shot-call_vewBy4KKSst6XTyViRcmqhTbfc_0fd14e2a9863ec86.png`

### 4.5 Accuracy statement

All API/behavior claims above are constrained to the verified code map in § header (`artifacts/bench-admin-code-analysis.md`) plus labeled clinic ops research (Sep 21–22) plus the **live UI walk** in § Live UI audit (22 Sep 2026 IST). Where the code map is silent, this document says **unknown**. Live observations that correct code-map framing (fleet UI present; Stop-all dominance; DEVICE_MISSING placement; waiting-unit confusion) are labeled **live** and do not invent APIs.

---

*End of deliverable — EvenScribe Designer, 22 Sep 2026 IST*
