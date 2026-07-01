# workout-gen — Continuation Brief

This is the handoff doc for resuming development. It is the single source of truth for
**how to work in this repo, what's done, and what's queued next**. Read it fully before
touching code. The owner values brevity, accuracy, and surgical changes — match that.

---

## 1. What this is
A personal, fatigue-aware training PWA with progressive-overload tracking. Single-page
React app, no backend, all state in `localStorage`. Deployed as a static site via GitHub
Pages. The owner trains with it directly, so correctness of the progression math matters
more than features.

Stack: Vite + React 18, no service worker. Files that matter:
- `src/App.jsx` — the whole app (~1150 lines). Almost all work happens here.
- `src/exercises.js` — 96-exercise catalog (each: `name`, `muscles`, optional `bw`, `hold`).
- `src/main.jsx` — entry. `index.html`, `vite.config.js`, `package.json` — standard.

---

## 2. How to work here (mandatory cadence — do not skip steps)
**One feature per commit.** For each change:
1. **Audit first.** `grep`/`sed`/`view` *every* place the change touches before editing.
   The owner distrusts broad/sloppy edits; trace data flow and all call sites.
2. **Edit surgically** (prefer `str_replace`; Python heredocs with asserts for multi-edits).
3. **Build:** `npm run build` — must print `✓ built`. (`pip`/`npm` allowed domains only.)
4. **Sanity-sim the logic** with a quick `node -e '...'` before trusting it.
5. **Commit** with a descriptive lowercase message (subject + short body explaining *why*).
6. **Push**, then **poll the GitHub Actions run** until `completed/success`.
7. **Confirm `git status --short` is clean** before starting the next feature.

**Deploy:** push to `main` → GitHub Actions → GitHub Pages (`dataguy99.github.io/workout-gen`),
~50–90s. Verify via:
`GET https://api.github.com/repos/DataGuy99/workout-gen/actions/runs?per_page=1` → read
`status`/`conclusion` of the top run.

**Push command:** `git push "https://<TOKEN>@github.com/DataGuy99/workout-gen.git" HEAD:main`
— the owner holds the personal access token and will provide it on request. **Never hardcode,
print, or commit the token.** (Do not paste it into any file in this repo.)

---

## 3. Owner preferences (hold these)
- **Brevity above all.** No fluff, platitudes, hedging, or "gassing up." Adult-to-adult.
- **Accuracy is non-negotiable.** Never compromise it. Verify before claiming.
- **Pragmatic, betterment-oriented** (no coping/indulgence framing).
- **Surgical & one-at-a-time.** Audit touchpoints, change one thing, verify, move on.
- **Hold pushes until an explicit go** ("go" / "build it" / "push that"). He was burned once
  by a push during an active session: a deploy/refresh can lose *unsaved React session state*
  (localStorage data is safe across deploys). Documentation/agreed-deliverable pushes are fine.
- **His example numbers are usually hyperbolic.** Use research-grounded values, not his literals.

**Training context (informs engineering decisions):** ~183–184 lb. Ramps a belt/landmine
squat (~150 lb). Uses pull-ups and nordics as bodyweight movements — hits e.g. 6 reps @ RIR 0
(clean failure), which is exactly the case the eccentric back-off logic handles. Runs in a
calorie deficit; uses MacroFactor for empirical burn. Body data trends waist up → the
composition verdict flags a fat-gain/recomp pattern.

---

## 4. Architecture map (where things live in `App.jsx`)
- **`SK`** = `localStorage` key map (`wg2-*`); helpers `sv()` / `ld()`. Stores: `anchors`,
  `anchorLog` (keyed `{exerciseName:[sessions]}`), `accLog` (array) + `accLog+"_prog"` (keyed
  accessory history — the one progression reads), `fatigue`, `prefs`, `banned`, `nutrition`,
  `body`, `cardio`, `meso`, `history` (session list, aka `sessHist`), `profile`, `daytargets`,
  `metgoal`, `eccentrix`.
- **Anchors:** 6 fixed `PATTERNS` (horiz/vert press, horiz/vert pull, squat, hinge). Each maps
  to one exercise; the "Change" button swaps the exercise. Progression via `getProgression()`.
- **`getProgression(name, log, repRange=[6,10], targetRIR=2, bodyWeight=0)`** (~line 263):
  routes bodyweight exercises (`exDef.bw`) → `bwProgression()`; otherwise loaded **"model C"**
  e1RM double-progression (Epley e1RM averaged across all sets). Call sites: `genAcc` (~391),
  anchor session gen `initSession` (~558), and inline in the anchor card render.
- **`bwProgression(ex, last, repRange, targetRIR, bodyWeight)`** (~line 212): RIR-band model.
  Holds → time progression. *Pure-clean phase:* RIR ≥ `RIR_PROGRESS`(3) add a clean rep;
  RIR 1–2 hold (optimal hypertrophy zone); RIR 0 (failure) backs clean reps **down** by
  `targetRIR` and converts the remainder to eccentrics; at the rep ceiling → add load.
  *Eccentric phase* (E>0, RIR read as eccentric reserve): ecc RIR ≥ 3 convert one ecc→clean;
  RIR 1–2 hold the mix; RIR < 1 ease toward more ecc; graduates back to all-clean. Gated by
  `eccEnabled` (`SK.eccentrix`). Research basis: Refalo 2023/24 & Robinson 2023 — hypertrophy
  zone ~1–3 RIR, failure adds fatigue w/o benefit, eccentrics ~40% stronger / less fatigue.
- **`SetRow`** component (~line 466): per-set inputs (weight/reps/RIR/pain). For `isHold`
  exercises it shows a start/stop **hold timer** that writes elapsed seconds into the reps
  field (~466–482). Shows an eccentric input when `showEcc`. **This is the component to extend
  for the per-set power timer.**
- **`genAcc(...)`** (~line 300s onward): headroom-aware accessory selector. Reads `accProg`
  (keyed history) once; recency penalty scaled by star rating; intensity cycling
  (8-12 / 12-15 / 15-20); `exclude` param prevents reroll dupes; `muscleSeedWeight()` seeds new
  accessories; deload reduces to 1 set @ 70%. Callers: `initSession`, `rerollAcc`.
- **`VOL_LANDMARKS`** = per-muscle `{mev, mav, mrv}` weekly hard-set targets.
  **`calcWeeklyVolume(anchorLog, accLog)`**: rolling 7-day window, counts hard sets (rir ≤ 4)
  with **fractional** muscle attribution across primary+secondary. (Known limitation: under-reads
  focused muscles due to fractional-vs-direct-set mismatch; not yet fixed — a 1.0 primary / 0.5
  secondary scheme was discussed but not implemented.)
- **`meso`** = `{startDate, length}` mesocycle; `mesoState.phase` drives deload behavior.
- **Trends helpers:** `slope(pts)` least-squares; `bodyTrend(entries, key)` fits a 2-var
  least-squares (value ~ day + hour) to control for time-of-day, but only at ≥5 measurements
  spread across varied times, else plain slope; `cardioBurn()` Keytel HR-based;
  `CARDIO_MET` (steady 7 / hiit 9 / rowing) fallback + `RESIST_MET` (5).
- **Tabs:** LIFT / LOG / TRENDS via `view===`. The **ECCENTRIC toggle** sits at the top of LIFT
  under the volume dashboard — **mirror this exact pattern for the new Power toggle.**

---

## 5. Done & deployed (latest commit `71dbea6`)

**Session 4 (2025-06-26) — all CI-green.** Double progression: reps climb 6→12 at a fixed load,
then load bumps & resets to the floor; a too-light load (RIR ≥ target+3) recalibrates INTO the range
— target reps are clamped to the ceiling, not kept above it (`99fb2dc`; clamp in `71dbea6`). RIR-gated
deload — only fires on a genuine 3-session volume drop at RIR≤1, which killed the false DB-row deload.
Anchor range widened 6→12. Per-exercise eccentric toggle replacing the global ON/OFF: in-memory
`eccBySlot`, an ECC button on each BW non-hold card; the toggle now re-seeds reps + ecc on the live
rows so they match the note (`ace69e4`; populate fix in `71dbea6`). MEV "hard sets" bar reworked to
show TYPICAL weekly volume — the average of recent COMPLETE weeks — so it reads full/green for a
consistent trainer instead of a low red stub early in the week; a tick marks this-week progress and a
per-muscle load-trend arrow shows tonnage vs last week (`c4a2626`, `2ae3cb3`). NOTE: this average-bar
approach SUPERSEDED the §8 pacing/cadence design — `cadencePace` + the pace constants were deleted as
dead code, and the advanced "lookback" setting was repurposed from learning cadence to the average
window.
Since `d7f00f7`: Trends-audit fixes — MEV hard-sets chart switched from a drifting Date.now()
rolling-7d window to the calendar week (`da25137`, also fixed the stray `$` in the POWER label
and unified the hard-set RIR<=4 rule for accessories); Sunday-start weeks to match owner's Sun–Sat
cycle (`a1f918e`). STILL OPEN from the audit: the two tonnage labels are swapped — section
"Weekly Tonnage" shows subtitle "Anchor volume load" (anchors only) while "Training Volume" shows
"Weekly tonnage" (anchors+accessories); now folded into the planned Lift-carousel + Trends consolidation (see Open follow-ups).
**ROADMAP COMPLETE — all of §6 (6.1 power, 6.2 set-count, 6.3 custom anchors) shipped.**
Session 3 (2025-06-23) also shipped, all CI-green: power timer on bodyweight rows
(`7b135e9`); per-set editable power timer + expiry alarm (`274ecdb`); anchor-change Done
button (`99743b4`); pain scoped to squat+hinge + anchor load auto-seed + removed duplicate
Strength Index (`19348ff`); power/eccentric toggle desync fix + new-anchor power routing
(`1d0bc1c`); power+eccentric default OFF each session (`915bdd9`); dynamic activeSlots
refactor (`89593c5`); custom-anchor configurator UI (`d7f00f7`).

**Open follow-ups (optional):** **[NEXT] Lift-carousel + Trends consolidation** — move all lift-only
metrics (MEV bars, ONE tonnage chart, Strength Index, Anchor Progression) into a swipeable carousel on
the LIFT page; strip them from TRENDS, leaving it a body/health overview (session history WITH
per-session values, weight/measurement trends, cardio, cross-domain insights), optionally embedding the
carousel mid-page. This one move resolves the swapped/duplicate tonnage charts, the MEV bar rendered on
BOTH tabs, and the empty tonnage "box" cards. Fold in while there: drop the Insights ≥4-aligned-weeks
gate (reads "insufficient" with only 2 weeks of data); add per-session values to history rows; the
"Avg N min" session stat blends quick+full. Cardio/MET pace-or-average bars (see §8) still unbuilt.
— Older items: custom-slot exercise picking uses curated PATTERN_MAP lists,
not full-catalog-by-category; anchor load auto-seed copies raw lb across exercises (no
per-implement vs total tagging); Power/Eccentric toggle regenerates the whole session
(rerolls accessories incl. locked) — could scope to anchors-only; configurator reorder uses
up/down (a drag grip could be added).

---
## 5b. Prior (commit `e5ca94c`)
**Session 2025-06-22 shipped (all CI-green):**
- `e45d84a` POWER (ballistic) mode — global toggle in LIFT mirroring ECCENTRIC (SK `wg2-power`,
  default OFF). Loaded anchors prescribe ~50% e1RM (plate-rounded, editable) × `POWER_REPS`(5)
  fast reps inside a `POWER_WINDOW`(15s) window; each loaded set row gets an optional countdown
  timer. Flat target: hit reps→+`POWER_INC`(5)lb, miss→hold. New `powerProg()`; `getProgression`
  gained a `powerMode` param (loaded anchors only; accessories + model-C untouched). Power sets
  tagged `pwr:1` and persisted so progression survives sessions. First power session derives load
  from e1RM, or 60% of top working set if no RIR history.
- `48417d0` CATALOG GAP-FILL — added 8 barbell/landmine staples with proportions assigned from
  biomechanics: Barbell Bench Press, Incline Barbell Bench, Barbell Push Press, Barbell Back Squat,
  Barbell Front Squat, Barbell Lunges, Barbell Curls, Landmine RDL. Anchor-relevant ones wired into
  PATTERN_MAP (fixes inability to pick a flat barbell bench for H.Press). Catalog now 104 exercises.
- `e5ca94c` ACCESSORY SEARCH + VARIETY FIX — search box under Accessories: type ≥2 chars, tap a
  catalog match → inserts as a locked accessory, search clears, unlocked slots auto-fill excluding
  locked names. Extracted `buildAcc()` (shared by manual-add + auto-selector). Fixed the "same ~8"
  funnel: softened the headroom term to a tiebreaker (`frac=0.5+0.5*head/cap`, was a hard gate) and
  built the pick window from top-`ACC_PER_CAT`(4) per movement category so push/pull/legs/core all
  surface (was: weighted-random over global top-6 = permanently calves/abs/shrugs). Recency 2→3.

**Prior (commit `01a248b` and earlier):**
- Accessory engine: recency + intensity cycling; fixed a bug where `genAcc` read the flat
  `accLog` instead of the keyed `accLog+"_prog"` (accessories never progressed); reroll dedup;
  star rating protects recency.
- Eccentric overload for bodyweight: per-set `ecc` input on dynamic bw rows; research-grounded
  RIR-band `bwProgression` (see §4); ECCENTRIC toggle (default ON) at top of LIFT.
- MET-hours/week Trends metric (editable goal, default 40; cardio + lifting split, stacked
  weekly bars + goal line).
- Removed vestigial activity-override row + day-target constants (nutrition target now solely
  `dayTargets[dow]`).
- Stripped bilateral limb measurements (both arms, both thighs) everywhere; kept weight/waist/navel.
- Removed redundant first→last body delta box (kept the slope box).
- Time-of-day adjustment for body trends via covariate regression (`bodyTrend`).
- Cardio: rowing type + distance (m) input + Z1–Z5 minute boxes (blank = 0). Commit `6837225`.
- Edit duration of saved lift sessions: in Trends → Session history each entry has an **edit**
  control that swaps the row for a minutes input (pre-filled), saves back by date, flows into
  the MET-hours resistance calc. Commit `01a248b`.

---

## 6. QUEUED — build next, one at a time, per the §2 cadence
Design is **already decided** for all three (owner delegated the exact rules where noted —
keep them research-grounded). Power is DONE (`e45d84a`). Remaining build order: Set-count → Custom anchors.

### 6.1 Power timers + power progression  — ✅ DONE, shipped `e45d84a` (see §5)
- **Global "Power" toggle** in settings, mirroring the ECCENTRIC toggle (new `SK` key e.g.
  `wg2-power`, default OFF). When ON, **every set row** gets an *optional* per-set timer —
  extend the existing hold stopwatch in `SetRow` (~466–482), which already does start/stop and
  writes elapsed seconds.
- **Power-set model (explicit owner decisions):** TIME is the constant the user sets (the
  window, ~15–20 s); the user logs REPS done in that window; **FLAT TARGET — no velocity-loss
  autoregulation.** Power is a third goal alongside hypertrophy and strength.
- **Prescription:** a submaximal load + a fixed target rep count to hit inside the time window.
  Load is exercise-dependent in the literature (jump/ballistic ~0–30%, bench ~30–45%, back squat
  ~56–65%, hex-bar DL ~60%, Olympic ~70–80% of 1RM; low reps at maximal velocity). The app
  doesn't classify exercises, so use **a single sensible default (~50% of current e1RM, rounded
  to plate, user-editable)** + a fixed target (e.g. 5 reps).
- **Progression rule (flat target):** reps_in_window ≥ target → add a small load increment next
  session (~+2.5–5 lb, or +2.5% e1RM-equivalent), keep target + time; reps_in_window < target →
  hold load. Implement as a new branch in `getProgression` (or a parallel `powerProgression`)
  keyed off the presence of timer data on the set.
- **Citations:** Cormie 2007 (jump squat 0%, squat 56%, power clean 80% 1RM); PMC9806758
  (back-squat Pmax 64.6%, hex-bar DL 59.6% 1RM); Wilson 1993 (30–60% trains force + velocity);
  reviews: submaximal 30–80% 1RM at maximal velocity, reps ~1–5 to preserve bar speed.

### 6.2 Set-count auto-progression  — ✅ DONE, shipped `63c1c24`
- Progression should adjust **set count**, not just reps/weight. Model: a weekly meso ramp —
  start near **MEV**, add a set once the rep range is topped out, climb toward **MRV** across the
  mesocycle, then deload. Use the existing `VOL_LANDMARKS` (mev/mav/mrv) and `meso`
  (startDate/length). New order of operations: **reps → add a set → add load.** Touches session
  generation (`initSession` set counts, `genAcc`) and the meso logic.

### 6.3 Custom-anchor configurator  — ✅ DONE, shipped `89593c5` (refactor) + `d7f00f7` (UI)
- Keep the fixed 6 as the **default** ("best"). Add a **smaller, separate** option to configure/
  select your own anchors and run that custom set for a mesocycle. Integrates cleanly: the
  volume/accessory engine already reads each anchor's *assigned-exercise muscle profile*, not the
  pattern label, and logged history is keyed by exercise name (so it survives slot changes).
- **SPEC LOCKED with owner (2025-06-22):**
  - At meso start, show TWO cards: a prominent **"Default anchor configuration"** (the fixed 6) and
    a smaller / duller / thinner **"Custom configuration"** card.
  - Custom = **variable slot count, 6–12 slots**. Each slot has a **pattern-type label** chosen from
    the 6 types (H.Press, V.Press, H.Pull, V.Pull, Squat, Hinge) — **repeats allowed** (e.g. two
    H.Press slots) — plus an **exercise of the owner's choice** assigned to that slot (allow ANY
    catalog exercise, ideally filtered/grouped by the slot's pattern category, NOT limited to the
    curated PATTERN_MAP).
  - Slots are **reorderable**. Owner wants a grip portion on each card to drag.
  - **Reorder engineering decision (made):** native HTML5 touch-drag is unreliable in this plain-React
    stack (no DnD lib available). Implement a **pointer-event drag grip** on each card, with **up/down
    arrow controls as a guaranteed fallback** so mobile never feels broken.
- **Implementation plan (blast radius audited 2025-06-22):** generalize the fixed `PATTERNS` const into
  a dynamic `activeSlots` array (default = the 6 templates; custom = saved config). Keep the 6 as
  pattern-type *templates* (id/label/full/muscles); a custom slot = `{id:"custom-N", type, label, full,
  muscles}` with the assigned exercise in `anchors[slot.id]` (unique id since labels repeat). Persist
  config under a new SK key (e.g. `wg2-anchorcfg` = `{mode, slots:[{id,type,exercise?}]}`).
  Then swap `PATTERNS` → `activeSlots` at all **13 sites** (so existing `anchors[p.id]`/`p.label`/
  `p.muscles`/`sets[p.id]` keep working unchanged): module fn `anchorMuscleLoad` (~378, change sig to
  take `slots` and thread to its 3 callers — `initSession`, `rerollAcc`, `addAccByName`), `allSet`
  (~598), `initSession` (~609), `saveSession` (~674, ~691 fatigue uses `p.muscles`, ~701 hist), anchor
  render (~872), anchor setup picker (~824, options should come from the slot's category not just
  `PATTERN_MAP[p.id]`), and the four Trends sites (~745, ~1078, ~1095, ~1127, ~1142). Logged history is
  keyed by exercise name so it survives slot changes. The configurator UI (variable slots, per-slot
  type + exercise pickers, add/remove 6–12, reorder grip) is ~100+ lines — build it as a dedicated
  component. Owner reorder UX: grip drag + up/down fallback.

---

## 8. SUPERSEDED — weekly pacing / encouragement bars
**Superseded 2025-06-26.** The "<MEV = red early in the week" problem this spec targeted was solved
instead by making the MEV bar show TYPICAL (average) weekly volume (`c4a2626`) — full/green for a
consistent trainer, no pace/cadence machinery. `cadencePace` was built (`7653e8e`) then removed as
dead code. The one genuinely unbuilt piece from this spec: **cardio/MET pace bars** (the spec applied
to cardio too) — now an open design question; a cardio "typical/wk" average would mirror the MEV bar.
Original spec kept below for reference only.

### Original spec (reference only)
## 8. PROVISIONED — weekly pacing / encouragement bars (NOT YET BUILT, spec locked)
Make the weekly bars (the MEV per-muscle "hard sets" chart + cardio + others) **pace-aware with
encouraging color**, replacing today's static "<MEV = red" (which paints everything red early in
the week even when nothing is wrong).

**Intent:** answer "am I on track for the week?" — GREEN at/ahead of pace OR within a grace margin
behind; AMBER when slipping past the margin; RED only when meaningfully behind. Bars carry it
(color + a "where you should be by today" pace tick), minimal/no text. Encourage, don't punish.

**Locked parameters:**
- Targets: current built-in landmarks (3 sets, 6–12 reps, RIR defaults / MEV). Tweakable later.
- Grace band: ~5–10% under target still GREEN (maintenance or a deliberate back-off when cooked are
  both fine); ~15–20% behind = flag. TUNABLE — dial once felt in use.
- Margin TIGHTENS toward end of week: 30% behind Sunday = nothing, 30% behind Friday = real. Early
  slack forgiven, late slack flagged → pushes a strong finish.
- Pace follows TRAINING cadence, not the calendar. A linear day/7 line would falsely flag rest days
  (owner rests ~Wed + Sat) as "behind." Pace only advances on days volume normally lands.
- Cadence is LEARNED from logged dates, PER MUSCLE (not declared, not per-day-global). Every
  workout/cardio is dated; infer per-muscle which weekdays / how much volume usually lands and build
  the expected pace curve from that. Owner does full-body x2/wk + cardio across 3 days now but wants
  it per-muscle so a future upper/lower or body-part split needs zero rework.
- Lookback window: DEFAULT 2 weeks (a 2-week change is probably a real regimen change; reverting =
  2 weeks the old way). Toggleable 2/3/4 weeks in a NEW ADVANCED SETTINGS tab (first resident of it).
  Shorter = snappier to flag/adopt a new pattern; longer = more forgiving.
- Applies to MEV per-muscle bars AND cardio/MET bars (all weekly-bucketed as of `da25137`).

**Cardio note:** cardio does NOT feed muscle MEV (different stimulus — a Z2 ride is not a bench set).
Cardio gets its own pace bars (kcal / MET / per-modality). If cardio-muscle-load is ever added it
feeds a SEPARATE conditioning/fatigue axis (the accessory engine's existing fatigue concept), never
the growth-volume bars.

**Foundation already in place:** calendar-week bucketing (`da25137`) + Sunday-start weeks (`a1f918e`)
— pacing only makes sense inside a correctly-bounded week. The MET-hours card (weekly load vs a
user-set goal, already blends cardio+lifting) is the closest existing pattern to start from.

**Open at build time (implementation, not spec):** exact shape of the tightening grace curve; how
"expected per-muscle volume by weekday" is derived from history (e.g. avg sets-per-weekday per muscle
over the lookback); whether to add a single "week on track?" headline above the per-muscle bars.

---
## 7. Notes / tunables
- Tunable constants the owner may revisit: `RIR_PROGRESS`(3), `RESIST_MET`(5), MET goal default
  (40), recency window (2 sessions), muscle-seed median.
- `bodyTrend` only engages the time-of-day regression at ≥5 varied-time measurements; otherwise
  it's a plain slope. It uses the log timestamp as a proxy for measurement time (best if the
  owner logs promptly).
- A commented-out `RAMP_PREFILL` block in `initSession` is intentional and harmless.
- **INFRA BUG (separate cleanup, not yet done):** `node_modules` is committed to the repo because
  `.gitignore` line reads `-e node_modules` (a stray `echo -e` artifact) instead of `node_modules`.
  CI runs `npm ci` so the committed tree isn't load-bearing. Recommend: fix `.gitignore` to
  `node_modules`, `git rm -r --cached node_modules`, commit. When working in this repo, stage ONLY
  `src/` files (the clone's `node_modules` churns on `npm install`); restore with
  `git checkout -- node_modules package-lock.json` before committing.
- An earlier 4-phase data spec was completed; **this brief supersedes it** for what comes next.
- When resuming: confirm the working tree is clean and `main` is at the latest commit, then start
  with §6.1, following the §2 cadence end to end.

## 9. PROVISIONED SPECS (locked 2025-07-01) — build order 1 → 2 → 3

### 9.1 Cardio duration as H:M:S  [BUILT this session]
Single "min" field replaced with three inputs H : M : S (colon display via `fmtDur`, e.g. `1:05:30`;
`M:SS` under an hour). `duration` stored as FRACTIONAL MINUTES (`h*60 + m + s/60`) so Keytel, rowing
pace, and MET-hours (all per-minute) are untouched. Old integer-minute entries render fine. Zones stay
in minutes.

### 9.2 Custom exercise with muscle involvement  [NEXT]
User defines a custom exercise: name · category (one of the 4 catalog cats: **pull / push / legs /
core**) · muscle-involvement rows. NO anchor-vs-accessory toggle — defined once, joins the full
`EXERCISES` pool, available BOTH in accessory search (by muscle) and the anchor-slot picker (by cat);
muscle-group definition is what drives volume regardless of where it's used. Muscle picker is a custom
overlay (DatePicker-style, no native dropdown) over the 12 groups (chest, back, shoulders — NO separate
delt heads — biceps, triceps, quads, hamstrings, glutes, calves, core, traps, forearms). Involvement is
a DISTRIBUTION that MUST sum to 100 (verified: all 104 catalog moves sum to exactly 100; each hard set
credits `pct/100` per muscle = 1.0 total, so 100 keeps every move's per-set volume comparable). UI shows
a live running total and normalizes to 100 on save (displaying the result, no surprise). Store all
involvement in `p` (`s:[]`) — volume math flattens `p+s`. Persist to a new SK key merged into EXERCISES
at load; shape `{name,cat,p,s:[],bw,eq,custom:true}`. Create + manage (edit/delete) live in the
accessory-search UI ("+ Add custom"; custom entries carry an edit/delete affordance).

### 9.3 Trends reorg — sections IN-TAB  [AFTER 9.2]
NOT the Lift-page carousel — that waits for the later overhaul. Reorganize the flat Trends scroll into
clear ordered sections with headers:
- **LIFT**: MEV bars, tonnage (collapse the two charts → ONE, fix the swapped label), strength index,
  anchor progression, lift sessions w/ per-session tonnage.
- **CARDIO**: kcal/week, MET-hours, HR trend, cardio sessions w/ duration + kcal + zones.
- **RECORDED**: weight trend, body measurements.
- **INSIGHTS**: body-composition verdict + cross-domain correlations (balance↔strength, cardio↔weight);
  relax the ≥4-aligned-weeks gate so they show sooner.
Leave the LIFT PAGE untouched for now (it still has its own MEV bar → the Lift/Trends MEV duplication is
knowingly deferred). Absorbs stragglers: tonnage dedup + swapped label, insight gate, per-session values.
FUTURE (later overhaul): a Lift page showing lift data by default (anchor segments + accessories at the
bottom) with a collapsible "insights" dropdown arrow; same per-domain dropdown for cardio and body. The
in-Trends sections are the stepping stone to that.
