# Changelog

All notable changes to the Exceed POC are logged here, newest first, with the
day they happened. This is a factual log of what changed and why — for
architecture, current known gaps, and how to build/test, see `HANDOFF.md`.

**Every future session that changes this project must add an entry here**,
dated (`YYYY-MM-DD`, today's date), before considering the change done — this
is a standing rule, the same way `HANDOFF.md` §6/§7 document other standing
rules. Put new entries at the top. A one- or two-line bullet per change is
enough; link to `HANDOFF.md`'s relevant section instead of duplicating
detailed rationale that already lives there.

## 2026-08-17 (continued — Zato-1's Eddie, Platinum, and a mechanised card-data audit)

The three items left open by the entry below, all closed. The roster is now
complete except Seth (excluded by standing user instruction).

- **Zato-1 is playable — the Eddie token system is built.** Eddie is a third
  board entity (`player.eddiePos`, `null` = out of play), owned by a player but
  explicitly "not a character and ignored when moving", so no movement code
  touches him. New engine section: `usesEddie`/`eddieInPlay`/`placeEddie`/
  `removeEddie`/`eddiePlacementSpaces`, plus `attackSourcePos`/`attackRange` —
  the latter replaces the bare `engine.range(attacker, defender)` in the Strike's
  range check and returns Eddie's distance for a Zato-1 Special/Ultra. His
  Innate places Eddie from a Normal's After clause (new character-level
  `normalsAfterHook`, a generator — the Normals-gated sibling of
  `attackAfterHook`); his Specials/Ultras hit for Advantage and remove Eddie at
  Cleanup unless the card says otherwise (`keepsEddieOnCleanup` — "Oppose" and
  Sun Void); his eight cards are transcribed from the workbook's `Zato-1` sheet.
  New `place_eddie` Strike yield → `place_eddie_choice` UI mode, plus a
  Boost-time `needsEddieSpaceChoice` → `eddie_space_choice` mode for Invite
  Hell; the board draws Eddie and the range readout gains a second figure for
  the distance that actually decides his Specials. AI: `chooseEddieSpace` scores
  each candidate space by how much of the hand reaches the opponent from there,
  and `evalMatchupDetail` takes an optional `eddiePositions` so the CPU's card
  scoring measures his Specials from Eddie too (omitting it reproduces the old
  behaviour exactly, which is what every other caller does).
- **Platinum the Trinity is playable.** Her Innate and Exceed are both
  "Cleanup: you may reveal the top card of your deck…", which had no extension
  point. Added a character-level `onStrikeCleanup` hook (a generator, called
  after the boost sweep and the card-level Cleanup clauses, so a boost it plays
  is not immediately swept), a one-shot `_sustainOnce` flag honoured by
  `cleanupBoosts`, and `engine.putContinuousBoostIntoPlay` factored out of
  `reviveBestContinuousBoostFromDiscard` (with an opt-in `runImmediate` for
  "PLAY that boost", which Sky High Claw deliberately does not use). Both
  prompts reuse the existing generic `named_choice_effect` yield — no new UI
  mode. Her Overdrive plays the ticked card's Continuous Boost.
- **The card-data audit is now a program, not a reading exercise:
  `test_carddata.js`** (+ `xlsx_reader.js`, a dependency-free .xlsx reader that
  unzips the container with zlib). It compares every implemented character
  against their sheet — Exceed cost, card names, Special/Ultra type, Gauge cost,
  RNG/POW/SPD/ARM/GRD, the Cancel column, printed Boost Force cost — and exits
  non-zero on any structural mismatch. It found **37 findings across the roster,
  all now fixed**:
  - **Four wrong Exceed costs**: Potemkin 2→**4**, Millia 2→**3**, M. Bison
    2→**3**, Dan 3→**2**.
  - **Three boosts that were free although they print a {1} Force cost**: Vega's
    Sky High Claw, Sagat's Tiger Uppercut, Dan's Gadouken.
  - **The Cancel mechanic was silently dead for 15 fighters.** The Season 4
    batch wrote `cancelable: true` at CARD level, but the engine only ever reads
    `card.boost.cancelable` (ui.js ×4, render.js ×1) — so 50 cards across chipp,
    ky, anji, may, ramlethal, nagoriyuki, baiken, faust, giovanna, ino, jacko,
    happychaos, goldlewis (and the new zato1) could never be Canceled and showed
    no Cancelable badge. All moved into their boost objects. Separately, 22
    cards on testament, potemkin, leo, millia and axl were missing the flag
    altogether although their sheets mark them 'C'; added. (Sol's seven already
    had it in the right place.)
  - Wording is deliberately not enforced — see the file's header comment for
    why, including the trap that printed "Critical:" clauses live in each card's
    `criticalText` field, not in `text`. A pass that "restored" them into `text`
    produced duplicate lines and was reverted.
- **`test_headless.js` gained `--focus=<characterKey>` and `--games=N`** (the
  same idea as `test_ai_vs_ai.js --focus`) and now prints every UI mode the run
  actually reached — the only way to tell a "0 crashes" pass that exercised a
  new mode from one that never entered it.
- Test coverage added: Zato-1 (33 assertions in `test_guiltygear.js` — range
  from Eddie, placement legality, Hit-Advantage, the Cleanup removal and its two
  exceptions, Invite Hell's recall-before-range-check, Leap's push from the
  attack's source, and the AI's placement heuristic) and Platinum (26 in
  `test_blazblue.js`, including one full Strike through `resolveStrikeSteps` to
  prove the cleanup hook is really wired in and not merely callable).
- Verified: `test_carddata.js` PASS (1 known exception — Happy Chaos's Deus Ex
  Machina, which is not a deck card); `test_guiltygear.js` 226/226;
  `test_blazblue.js` 136/136; `test_changecards_guile.js` 19/19;
  `test_cammy_zangief_fix.js` 16/16; `test_ai_planner.js` 39/39;
  `test_onboarding.js` 90/90; `test_headless.js` 0/200 crashes (plus a focused
  `--focus=zato1 --games=40` run, 0 crashes, reaching both Eddie modes);
  `test_ai_vs_ai.js` 100/100 finished, 0 crashes. Not verified in a browser.

## 2026-08-17 (continued — fighter photos for Sagat and Ky Kiske)

- **Fighter photos for Sagat and Ky Kiske**, extending the `FIGHTER_PHOTOS`
  mechanism in `render.js`. Both source files (`sagat.png` / `ky kiske.png`)
  were already sitting in the project folder at 1024x1536, 24bpp RGB (no
  alpha). Checked both for the bottom white-strip defect documented in
  `render.js`'s `FIGHTER_PHOTOS` comment first — pixel-sampled the bottom 15
  rows of each; neither had it. Resized to the usual 480px wide (1024x1536 ->
  480x720) via .NET GDI+ (System.Drawing, high-quality bicubic) and re-encoded
  as JPEG at quality 88, same as the rest of the photographic-content batches
  — sagat.jpg 109,572 bytes, ky.jpg 129,027 bytes. Keyed by `sagat` and `ky`
  per `CHARACTERS` in `cards.js`. Verified by extracting both data URIs back
  out of the built `exceed_poc.html` and decoding them: `image/jpeg`, correct
  byte counts, and 480x720 decoded pixel dimensions for each.
- Verified: `node build.js` + `node -c bundle.js` clean; `node -c render.js`
  clean; `test_headless.js` 0/200 crashes; `test_blazblue.js` 136/136;
  `test_guiltygear.js` 226/226.

## 2026-08-17 (latest — Rulebook-correct Change Cards, Guile finished, card typing fixed)

Started as "implement the unfinished fighters"; most of it turned into correcting
mechanics that were wrong for characters already shipping. Two source documents
settled questions this project had previously recorded as undecidable: the
Exceed Comprehensive Rulebook (found on disk at `D:\stažené\Exceed_ The
Comprehensive Rulebook.txt`) and `EXdata.xlsx` itself.

- **Two stale claims in `HANDOFF.md`, corrected by checking the source.**
  (1) "`EXdata.xlsx` has no Zato-1 sheet at all, so there is no card data to
  transcribe" — the workbook has a complete `Zato-1` sheet, including the Eddie
  rules text; he is blocked only on the Eddie token system, not on data.
  (2) Guile's Innate was recorded as "genuinely ambiguous without the source
  rulebook" — the rulebook resolves it exactly (below).
- **Change Cards now follows the rulebook.** It is defined there as "spend Force
  to draw an equivalent number of cards", with Force generated by discarding —
  Normal/Special 1, **Ultra 2**, **Gauge card 1**. This engine implemented it as
  "discard N cards from HAND, draw N", duplicated in the human path and the AI
  path, so Gauge could never be spent. New `engine.performChangeCards()` is the
  single implementation, paying through the existing `payForceFromHandOrGauge`
  exactly like Move/Boost/character actions; both call sites now use it, the
  picker offers hand + Gauge, and the panel shows the projected draw live
  (spending one Ultra draws 2).
- **Guile is finished and no longer marked unfinished.** His Innate
  ("when using a Change Cards action, the first Gauge spent generates an extra
  {2}") is `CHARACTERS.guile.changeCardsForceBonus` — a flat +2 once per action,
  since the text says the FIRST Gauge card, not each one. His Exceed's "you may
  then Strike (after drawing cards)" is `mayStrikeAfterChangeCards`, offered
  through the existing `may_strike_choice` mode rather than auto-Striking. The
  AI spends a Gauge card during Change Cards only when playing a character with
  that bonus, since for everyone else Gauge and hand cards are both worth 1
  Force there and Gauge has better uses.
- **Card typing was systematically wrong for 16 characters (78 card
  definitions).** Their five signature cards were typed `'normal'` (a type that
  should only ever describe the shared blue pool), and every Gauge-cost card was
  typed `'special'`. The second half was a real gameplay bug: `payUltra` gates on
  `type !== 'ultra'`, so **Guile's Flash Explosion [4], Vega's Bloody High Claw
  [3]/Splendid Claw [3], M. Bison's Nightmare Booster [3]/Psycho Punisher [3],
  Sagat's Tiger Cannon [4], Chun-Li's Hosenka [4], Axl's One Vision [4] and
  others cost no Gauge at all to Strike with**, and were worth 1 Force instead of
  2 when spent. The first half mis-scoped every "your Normals ..." passive
  (Axl's `normalsStatMods`, Potemkin/Sol's Normals hooks) onto the character's
  own Specials. Fixed line-by-line, anchored on `owner:`, with each change
  printed for review. Season 5's three-Ultra characters and Dan's/Bang's extra
  Ultras were checked against the workbook and are legitimate (Astral Heat).
- **Three data bugs found by a card-data audit against the workbook**:
  Akuma's `exceedCost` was 3, the sheet says **2**; Guile's Flash Explosion boost
  used `sustainIfNoNormalUsed` when its printed Cleanup clause is "if you were
  **not stunned**" (wrong in both directions — new free `sustainIfNotStunned`
  flag, sibling of Vega's existing paid variant); and Chun-Li's Head Stomp
  carried an invented `range: [1,1]` although the sheet prints RNG **X**, i.e. no
  band at all. Head Stomp now has `range: null` plus a new card-level
  `rangeIncludeIfSideSwapped`, so its only way to connect is the printed
  "if you moved past the opponent" clause — which also un-defers that clause.
- **`test_headless.js` harness fix**: its `changecards_select` case picked from
  hand only, so once the pool included Gauge one game in 200 could never satisfy
  `sel.n` and hung in that mode. Caught by the suite itself.
- **`test_changecards_guile.js` (new)**: 19 assertions covering Force-based
  Change Cards, the Gauge payment path, Guile's once-per-action bonus, his
  Exceed's follow-up Strike through the real UI flow, and a roster-wide guard
  that no character card is mistyped again.
- **Verification**: `test_changecards_guile.js` 19/19; `test_headless.js` 200/200
  games 0 crashes; `test_ai_vs_ai.js` 100/100 0 crashes; `test_guiltygear.js`
  185/185; `test_blazblue.js` 110/110; `test_cammy_zangief_fix.js` 16/16;
  `test_ai_planner.js` 39/39; `test_onboarding.js` all checks pass. Not verified
  in a browser (no browser tooling available this session).
- **Still open**: Zato-1 (needs the Eddie token system — data is ready),
  Platinum the Trinity (needs a character-level Strike-cleanup hook — data is
  ready), and a full card-data audit of the remaining Street Fighter, Guilty
  Gear and BlazBlue cast, which was started and interrupted. Seth remains
  excluded per standing user instruction, re-confirmed this session.

## 2026-08-17 (Beginner onboarding: mechanics-driven, not text-driven)

Whole-session feature. Premise, from the user: the How to Play text is weak, and
onboarding should follow the game's own mechanics rather than paraphrase that
page. So nothing here adds rules prose — every new explanation is generated from
the state the engine is actually in, or from card data.

- **`engine.js` — `lastStrikeInfo`**: `resolveStrikeSteps` now records each
  exchange as structured data alongside the prose log: both sides' printed vs.
  live stats (after Criticals/Boosts/EX/face-up/character passives are folded
  in), who activates first, then ordered events — `activate` (the range check,
  recorded at activation time so the post-Before-movement distance is what gets
  shown), `damage` (effPow/effArm/effGrd/dealt, with Block's reactive Armor
  already folded in), `stun` (including the no-stun and stun-immune cases),
  `cleanup` (every disposal branch reports its destination: gauge/discard/
  sealed/deck/inplay), and `next` (who acts next, and whether Advantage caused
  it). Initialized to `null` in the constructor — never `undefined`. Nothing in
  the game logic reads it; it is a rendering feed, like `fxQueue`.
- **`ui.js`**: `GameController` takes an optional 4th `opts` arg
  (`{beginner:true}`) — every existing 3-arg call site is unaffected (HANDOFF
  §8 #12). Beginner mode gates the Critical prompt until the player has both
  dealt and taken damage (`criticalPromptUnlocked`/`noteBeginnerDamage`;
  declining is the no-op answer, so skipping the prompt cannot change game
  state), paces the strike breakdown (`beatIndex`/`revealNextBeat`/
  `revealAllBeats`), and keeps coach tips expanded. New `suggestMove()` runs
  ai.js's `chooseAction` on the human's own position at `randomness: 0` and
  turns the result into a sentence that says why — the same evaluator the CPU
  uses, so the advice cannot contradict the rules. Coach-tip persistence
  (`coachSeen`/`markCoachSeen`/`coachExpanded`/`toggleCoach`) lives here, not in
  render.js, which must stay pure; "seen" is marked when the player LEAVES a
  mode rather than on first paint, because several modes repaint repeatedly
  while the player is still deciding. localStorage access is wrapped with an
  in-memory mirror (the headless stub has none; real browsers can throw).
- **`render.js`**: `MODE_HELP` (22 modes) renders a `.coach-panel` explaining
  the decision the engine is asking for right now — keyed on `G.mode`, so it
  cannot drift from the real decision points. `TERM_HELP` is now the single
  glossary behind every tooltip (stat abbreviations, Gauge/Force cost badges,
  the Cancelable clock, the life-panel resource row), and `boldCardKeywords`
  wraps each `Before:`/`Hit:`/`After:`/`Now:`/`Cleanup:`/`Critical:` header in a
  `.term` tooltip. Board cells show their distance from the human
  (`.cell-dist`, `.band-self`); hovering a card in strike/defend lights that
  card's range band via guarded direct DOM class toggling (`highlightBand`/
  `clearBand` — deliberately NOT a re-render, which would kill both the hover
  and the FLIP animation), and each card carries an in-range/out-of-range chip.
  `strike_result` replaces "Strike resolved / Continue" with a 7-beat breakdown
  built from `lastStrikeInfo`, revealed one beat at a time in beginner mode.
- **`shell.html`**: the How to Play button and overlay moved OUT of `#app` so
  they survive `render()`'s repaint — the rules are now reachable mid-game,
  which HANDOFF §1 had flagged as a trap. Added Quick Start (Ryu vs Ken,
  Student, beginner mode), a three-line card-economy note (hit → Gauge, miss →
  discard, Force is paid by discarding cards), a "Recommended for beginners"
  roster section (ryu/ken/cammy/zangief) with the rest behind a toggle and
  unfinished fighters last, and a difficulty screen rewritten in
  player-experience language ordered easiest-first (the AI-internals paragraph
  is now a collapsed `<details>`). `startGame(difficultyKey, opts)` passes opts
  through. All new CSS lives here.
- **`test_onboarding.js` (new)**: scripted (not random) coverage of what no
  other suite touches — `lastStrikeInfo`'s recorded shape and field types, the
  coach expand/collapse rule under a no-localStorage environment, the beginner
  Critical gate, beat pacing, and `suggestMove()`. ~90 checks. Note its check
  COUNT varies run to run (it plays a real unseeded hand); only pass/fail matters.
- **Verification**: `build.js` clean; `test_onboarding.js` all checks pass;
  `test_headless.js` 200/200 games, 0 crashes (this is the suite that fails on a
  literal `undefined`/`NaN` reaching rendered HTML — the main risk in work like
  this); `test_ai_vs_ai.js` 100/100, 0 crashes; `test_guiltygear.js` 185/185;
  `test_blazblue.js` 110/110; `test_cammy_zangief_fix.js` 16/16;
  `test_ai_planner.js` 39/39; `test_ai_difficulty.js` 0 crashes, both ladder
  checks `true`. **Not verified in a real browser** — no browser automation was
  available this session, so the new CSS (coach panel, breakdown tables, band
  highlighting, Quick Start/roster layout) was checked only by rendering
  headlessly and cross-checking every emitted class against the stylesheet.
  That cross-check did catch two real defects (`.stat-compare` was styled as a
  `<table>` while render.js emits divs; `.stat-compare-label` had no rule at
  all), so treat one manual browser pass as still owed.

## 2026-08-17 (Effects iconography guide added to How to Play)

- **`shell.html`**: new third How to Play tab, "Effects iconography", next to
  "New player intro" and "Complete keyword glossary". `HOWTOPLAY_ICON_GROUPS`
  (hand-authored, same pattern as the existing `HOWTOPLAY_KEYWORDS`/
  `HOWTOPLAY_FGC_TERMS` arrays) groups the 21 fx `kind`s the FX layer actually
  emits today into "on the board during a Strike" (hit/miss/stun/move/armor/
  damage/ex — the ones tied to a board position) vs. "everywhere else — the
  fight log" (draw/discard/gauge/force/boost/etc.). Each entry's glyph is read
  live via `fxIcon(kind)` (engine.js) rather than hardcoded in shell.html, so
  the guide can't drift from `FX_ICONS` the way a hand-copied glyph could —
  only the label/description text is hand-authored. New `.htp-icon-list`/
  `.htp-icon-glyph`/`.htp-icon-note` CSS, same emoji-font fallback as the
  board badges.
- **`engine.js`**: added a comment on `FX_ICONS`'s `wildswing`/`reveal`/
  `critical` entries noting they're reserved extension points no call site
  actually tags yet (confirmed via grep across `engine.js`/`ui.js` — 21
  distinct kinds are genuinely emitted) — the guide deliberately excludes
  these three so it doesn't promise players an icon they'll never see.
- **Verification**: rebuilt via `build.js`, `node -c bundle.js` clean. Since
  `verify_howtoplay.js` doesn't exist in this project snapshot (confirmed —
  see HANDOFF.md §2's note on this), wrote a one-off Node `vm` smoke test
  instead: loaded the built `exceed_poc.html`'s four `<script>` blocks into a
  sandboxed context and called `showHowToPlayTab('icons'|'intro'|'keywords')`
  directly — all three render real HTML (icons tab: 4916 chars, contains the
  new tab label and at least one glyph entry) with zero literal `undefined`
  leaks. `test_headless.js` re-run afterward: 200/200 games, 0 crashes.

## 2026-08-17 (graphical FX layer: icons, board badges, paced playback, fluent stand movement)

Added the graphical-effects layer requested for the 9-tile board: emoji
icons for every strike/economy event, floating board badges tied to a
fighter's cell, step-by-step paced playback of a resolved exchange (a beat
per event instead of the whole thing landing at once), and fluent stand
movement via FLIP (the tokens now visibly slide between cells instead of
jump-cutting, despite `render()`'s full-innerHTML-replace design). No
Python involved — this is all browser JS/CSS at runtime; the one place
Python appears in this project (the old build snippet in HANDOFF.md §2) has
had a Node equivalent, `build.js`, for a while.

- **`engine.js`**: new `FX_ICONS`/`fxIcon()` glyph table and `this.fxQueue`
  (parallel to the existing text `this.log`). `logMsg(msg, kind, who, extra)`
  gained three optional args — when `kind` is passed it both prefixes the
  matching emoji onto the log line and pushes `{kind, who, ...extra}` onto
  `fxQueue` for board-badge/playback rendering. Existing 2-arg-or-fewer call
  sites are untouched; roughly 60 call sites across strike resolution
  (hit/miss/stun/critical/move/EX), boosts, and economy actions
  (draw/discard/Gauge/Force/Exceed/Revert/seal/heal/Overdrive/sustain/
  Advantage) now pass a kind. `ui.js`'s `log()` wrapper forwards the same
  three args.
- **`render.js`**: `boardHTML()` tokens gained `data-pid="p1"/"p2"` and a
  floating `.fx-badge` (icon + terse amount) drawn over whichever fighter's
  cell `G.currentFx` points at. The old `render()` was renamed to `paint()`
  (still the exact same full-innerHTML-replace body); a new `render()`
  dispatcher decides whether to paint immediately or hand off to
  `G.playFx()` first when `engine.fxQueue` is non-empty, and `paintWithFlip()`
  wraps every paint with FLIP (First/Last/Invert/Play): capture each
  `.token`'s screen position before the repaint, animate `translateX` back
  to zero after. Guarded via `captureFlipPositions()`'s `!document.
  querySelectorAll` check (same style as the existing `applyPendingShake`
  classList guard) so the headless test DOM stub is unaffected.
- **`ui.js`**: `GameController` gained `fxPlaying`/`currentFx`/`playFx()`/
  `skipFx()` — deliberately plain fields, not a new `G.mode`, so
  `test_headless.js`'s `Unhandled mode:` driver-coverage check (HANDOFF §5)
  doesn't need a new case. `playFx()` steps through `fxQueue` one event at a
  time (780ms hold on a `hit`, 520ms otherwise), re-rendering between each;
  under the headless DOM stub (no `querySelectorAll`) it drains the whole
  queue in one paint instead, so the 200+-game suites stay fast. New
  `scheduleAITurn(delay)` replaces all 7 `setTimeout(() => this.runAITurn(),
  N)` call sites — it waits out any in-flight `fxPlaying` before actually
  calling `runAITurn()`, so the CPU doesn't visibly start its next turn
  while the human's last exchange is still playing out on the board.
- **`shell.html`**: `.fx-badge`/`.fx-skip-btn` CSS, `.token{position:
  relative}` + `.board`/`.cell{overflow:visible}` so badges can float above
  a cell without being clipped, and an emoji-font fallback
  (`"Segoe UI Emoji","Apple Color Emoji","Noto Color Emoji"`) on both
  `.fx-badge` and `.log-panel` for glyph coverage on Windows/Chrome. A
  "Skip ⏭" button appears next to the range readout while `fxPlaying`.
- **Verification**: rebuilt via `build.js`, `node -c bundle.js` clean.
  `test_headless.js` 200/200 games, 0 crashes (the headless-drain path in
  `playFx()` kept this fast — no real `setTimeout` waits). `test_ai_vs_ai.js`
  100/100 games, 0 crashes. `test_ai_planner.js` 39/39,
  `test_guiltygear.js` 185/185, `test_blazblue.js` 110/110,
  `test_cammy_zangief_fix.js` 16/16 — none of these touch rendering, run as
  a regression check that the `logMsg` signature change didn't break any
  message-content assertion. `test_ai_difficulty.js` run for its usual
  ladder-trend check (unaffected by this change — no engine-logic edits,
  only render/UI/log-signature ones).
- **Known scope limits, left honest rather than silently claimed done**:
  board badges only carry real positional/amount data for the strike-
  resolution and movement events (`hit`/`miss`/`stun`/`move`/`ex`) — the
  ~50 economy events (draw/discard/Gauge/Force/boost/etc.) get the emoji
  prefix on their log line but no board badge, since they're not tied to a
  board position in any way that would read better than the log. Not
  every one of the ~200 individual `logMsg` call sites in `engine.js` got a
  `kind` — the ones left plain are structural (round headers, HP summaries)
  or informational text that would double up with an adjacent iconified
  line (e.g. Sickle Storm's "will Strike with X" right after its own
  discard line already got one). If a future card's flavor text needs its
  own icon, add a matching `FX_ICONS` entry and pass `kind` at that one
  call site — the plumbing doesn't need to change.

## 2026-08-17 (F-12 modelling gaps: Force pricing, dead code, two report claims retired)

Continues the teardown follow-up below. Two items picked up from that report's
ranked list turned out to be **wrong or already fixed in the source**, recorded
here so nobody re-chases them:

- **`scoreCharacterAction` was never "9 of 40 characters".** The report claimed
  31 characters fall through to `-Infinity` and so never use their character
  action. They don't have one: only **9 of the 40 entries in `CHARACTERS` have an
  `action` at all**, and they are exactly the 9 the old branch chain covered.
  `-Infinity` is correct for the rest — `render.js`'s `actionButtons()` doesn't
  draw a button for them and `ui.js` never offers it. The `planning`-object
  refactor (each action's scoring metadata declared in `cards.js`, read
  generically by `ai.js`'s `scoreCharacterAction`, with `scoreCharacterActionLegacy`
  as the fallback) is therefore an **architecture fix, not a coverage fix**.
  `_planning_inventory.txt` documents all 40 characters and the judgement calls.
  That refactor landed earlier but was never logged — this bullet is its missing
  entry.
- **"Ranges named: 0 across 100 games" is not dead code.** `chooseRangeToName`
  fires on `boost.needsRangeNameChoice`, which exists on **exactly one card in the
  game**: Zangief's Flying Power Bomb (`cards.js`). Zero firings means that card
  was never boosted in the sample, not that the path is unreachable. No fix
  needed; don't spend another session on the counter.

Actual changes:

- **The planner priced movement in cards, not Force** (F-12). `planMoveCost`
  returns one per space — which is Force — but its result was compared against,
  and subtracted from, raw hand+Gauge **card counts**. An Ultra pays 2 Force
  (`engine.js`'s `forceValueOfCard`), so a hand holding one could afford moves the
  planner ruled out, and paid for the ones it took with fewer cards than the search
  deducted. Fixed by giving `planRootState` a `forcePerCard` average (computed once
  at the root over hand+Gauge, the same resolution as `multiUse`) plus
  `planAvailableForce`/`planCardsForForce` helpers. Force is **derived from the card
  counts** rather than tracked as a second field, so the many places that adjust
  hand size cannot silently desynchronise it.
- **Paid character actions were free in the plan.** Found while fixing the above:
  `planApplyMyAction`'s `action` branch never deducted `actionForceCost`, so
  Baiken's/I-No's/Jack-O's `[1]` cost nothing in the search. Now deducted through
  the same Force→cards conversion. (`planCandidateActions` already gated
  affordability correctly; only the spend was missing.)
- **`shouldWildSwing` removed.** It ran *after* `wildSwingExpectedValue`'s
  principled comparison inside `chooseStrikeSelection`, so its
  `bestHandScore < -1.5` threshold plus flat 15% dice roll could override a
  decision that had just weighed Wild Swing properly and declined. A comment at the
  old site records why it must not come back.
- **`chooseStrikeCard` deleted** — the original Monte Carlo picker, superseded by
  `chooseStrikeSelection` (EX pairs + a solved mixed strategy), with no caller
  anywhere in the project. It was still exported and still being maintained. Its
  stale entry in `test_headless.js`'s `vm` exposer went with it.
- Verified after all four changes, on a freshly built bundle: `node build.js` +
  `node -c bundle.js` clean; `test_ai_planner.js` 39/39; `test_guiltygear.js`
  185/185; `test_blazblue.js` 110/110; `test_cammy_zangief_fix.js` 16/16;
  `test_headless.js` 0 crashes/200 games; `test_ai_vs_ai.js` 0 crashes/100 games;
  `test_ai_difficulty.js` 0 crashes across all 2100 games, **both ladder checks
  `true`** (vs-Dumb 90.7% / 90.3% / 78.7%).

**Two caveats for whoever picks this up — neither is a regression, both are
things not to over-read:**

- **The Force fix is NOT A/B measured.** `test_ai_strength.js` has no arm for the
  old card-count pricing, and per the teardown report's own closing argument
  self-play cannot settle it regardless. It is justified as a **correctness** fix
  (the planner compared Force against card counts, and an Ultra pays 2 Force),
  not as a demonstrated win-rate gain. If someone wants the measurement, the arm
  to add is one that forces `forcePerCard` to 1.
- **Champion vs Warrior read 58.3% here against 62.7% in the entry below.** At
  N=300 that gap is ~1.5 standard errors — noise, not a regression, and the
  ladder's own tolerance checks passed. Recorded because it is a *drop*: don't
  quote 62.7% as the current figure, and don't treat a 58% reading next session
  as evidence something broke. Same for `test_ai_vs_ai.js`'s out-of-strict-range
  strikes, 23.6% here vs 28.1% before — suggestive, well inside noise at n=100,
  and NOT claimed as an improvement.

## 2026-08-17 (CPU teardown findings verified, measured and closed out)

Context: a preceding session audited the CPU against expert play and produced a
12-finding teardown report (F-01…F-12, ranked, with a Tier 0–5 roadmap), then
implemented most of Tier 0–2 in `ai.js` — and ended without running a single
test, taking a single measurement, or writing a changelog entry. This session
verified that work, measured it against the report's own KPIs, and finished the
two Tier 0 items still open. **The headline is a negative result: the fix the
report predicted would be "the largest single jump" measurably isn't one.**

- **Verified the previously-unverified work** (all green, on a freshly built
  bundle): `test_ai_planner.js` 39/39; `test_guiltygear.js` 185/185;
  `test_blazblue.js` 110/110; `test_cammy_zangief_fix.js` 16/16;
  `test_headless.js` 0/200 crashes; `test_ai_vs_ai.js` 0/100 crashes;
  `test_ai_difficulty.js` 0 crashes, both ladder checks `true`.
  Which findings had actually landed, confirmed by reading the source rather
  than trusting the report: F-01 (Gauge priced via `GAUGE_POINT_VALUE`, plus an
  Exceed reserve in `ui.js`'s `aiDecideCritical` mirroring `aiShouldCancel`),
  F-03, F-04 (`buildBoostCandidates` — the planner now picks the card too),
  F-05 (`solveMixedStrategy`, regret matching, wired at `ai.js`'s
  `chooseStrikeSelection`), F-06 (`evalMatchupDetail` returning a real damage
  record), F-07 (`certainLethal`), F-09 (`EX_DISCARD_COST`), most of F-10, and
  the first step of F-11 (observation-weighted opponent sampling).
- **Re-measured the report's KPIs** over 100 instrumented games. F-03 is
  confirmed fixed with exactly the predicted signature: **Prepare 0 → 254
  actions**, the discounting asymmetry gone. F-04 moved its markers too (Focus
  Readings 11 → 18, Block names 9 → 11).
- **But Tier 1's stated KPI did not move.** Exceed was legal at 731 decisions
  and taken 106 times (**14.5%**, against a 15.9% baseline); average Gauge at
  decision time 1.77, against 1.73. Roughly half of all players still never
  reach their Exceeded half. Pricing Gauge did not change the behaviour the
  report said it would.
- **Diagnosed as arithmetic in `planEvaluate`, then A/B'd — and the obvious fix
  is not an improvement.** Exceeding pays `gauge` × exceedCost (0.35 × 2) plus
  the `canExceed` threshold bonus (1.0) and spends the turn (handing the
  opponent `oppTempo`, 0.5) to gain `exceeded` (2.5): net ≈ +0.3, which loses to
  almost any real Strike. Raising `w.exceeded` was measured with
  `test_ai_strength.js` at 100 paired seeds (200 games) per arm:
  **4.0 → 49.0% [43.6%, 54.4%]; 5.5 → 48.0% [42.6%, 53.4%]** — both intervals
  straddle 50/50 and both trend slightly negative. **Not adopted; the weight is
  unchanged.** The two configs are kept in `test_ai_strength.js` as
  `exceedhungry`/`exceedhungry2` so the next session re-runs rather than
  re-guesses. Standing caveat, which is the report's own closing point: this is
  self-play, so it cannot detect a blind spot both sides share.
- **F-08 fixed (real bug, rules-level).** `resolveAIStrikeRequest`'s
  `block_force` spend was `ceil((POW − ARM − GRD) / 2)`. Guard does **not**
  reduce damage — `engine.js` computes `dealt` from Armor alone and uses
  `effGrd` only to gate the Stun — so that expression is the *stun-avoiding*
  spend, always the smaller of the two, and it was being used as the
  damage-preventing one. With any Guard at all the AI under-spent Force and ate
  damage it could have prevented. Now both objectives are computed separately
  (`toZeroDamage`, `toAvoidStun`), the larger is taken, and a 2-Force hand
  reserve applies to everything above the stun-avoiding amount. Measured effect:
  Block-force spends 53 → 62 per 100 games. (The engine half of F-08 —
  `forceValueOfCard` instead of flat 1-per-card, cheapest-first selection, and a
  `uids` picker — was already fixed in the preceding session.)
- **F-02 closed as a deliberate decision, not a deletion.** The ~90 lines of
  one-turn heuristic under the planner stay, because `test_ai_strength.js`'s
  `legacy` config flips `PLAN_CONFIG.enabled` off to play the planner against
  them — deleting the dead code would delete the only evidence the planner earns
  its cost. They now carry an unmissable `DO NOT TUNE ANYTHING BELOW THIS LINE`
  banner recording that `planThreeTurns` returned a plan 400/400 times, listing
  exactly which tuned-looking knobs are unreachable, and pointing future work at
  `PLAN_CONFIG`/`planEvaluate` instead.
- Ladder is also visibly sharper than the historical figures: Champion beats
  Warrior **62.7%** head-to-head (this gap used to disappear into noise near
  50/50 — see `HANDOFF.md` §9's "difficulty ladder noise" pitfall), Champion vs
  Student 74.7%, vs-Dumb trend 93.3% / 90% / 77.3%.
- Verification after the two fixes: `node build.js` + `node -c bundle.js` clean;
  all four assertion suites green as above; `test_headless.js` 0/200;
  `test_ai_vs_ai.js` 0/100; `test_ai_difficulty.js` both checks `true`;
  `test_ai_strength.js` self-mirror exactly 50.0% (harness sound) and
  `--baseline` current-vs-weak 83.3% [77.3%, 89.3%].
- Still open from the report: F-11 beyond its first step (a weighted particle
  filter with resampling), F-12's minor modelling gaps (crossing invisible to
  `evalMatchup`, Move cost counted in spaces not Force, inert `shouldWildSwing`,
  dead `chooseStrikeCard`, `scoreCharacterAction` covering 9 of ~40 characters),
  and Tiers 4–5 (ISMCTS over a snapshot-able engine; automated weight tuning).

## 2026-08-17 (later still — three-turn AI lookahead)

- **The CPU now plans three turns ahead** (my turn → the opponent's answer → my turn)
  instead of scoring only the current one. New `PLAN_CONFIG`/`planThreeTurns` section in
  `ai.js`; `chooseAction` calls it at its decision point and returns the first action of
  the best line. The existing one-turn heuristic was NOT rewritten — it still runs
  underneath as the fallback, and `chooseAction({..., usePlanner: false})` selects it.
- Design: every Monte Carlo sample is drawn ONCE per decision into two memoized tables
  (my best exchange at a distance; the mirror from the opponent's side), and the search
  is pure arithmetic over them — so depth costs lookups, not nested sampling. That is
  the direct answer to `HANDOFF.md` §6's long-standing argument against building this;
  that bullet is now updated rather than just deleted.
- The evaluation prices Range, Gauge efficiency, hand size vs the character's hand
  limit, continuous Boosts, Exceed Mode, Advantage, deck-out risk, multi-use cards and
  Strike setup — all in the same net-damage units, so they trade against each other
  honestly.
- **Three bugs found by measuring rather than by reading**, each worth real win rate:
  - the opponent's turn was scored with MY defensive value, which is positive when I
    defend well — so "stand where they can hit me" read as a bonus. Now priced as minus
    what the turn is worth to THEM, clamped so it can never be positive. 47% → 55%.
  - damage was counted twice on two different scales (as an immediate gain AND through
    life at the leaf), which collapsed the planner into a greedy striker — 57% of all
    turns were Strikes. Damage is now priced once, via the `life` weight.
  - Striking was free: the exchange table always reports the root hand's best attack, so
    a plan that Struck twice was credited with its best card twice (`usedAttacks` decay).
- Tuning attempts that measured WORSE and were reverted: 8 opponent-hand samples
  (55% vs 56% and slower), 2 opponent responses per sample instead of 3 (51.3%),
  crediting a Boost's `estimateBoostTimingValue` on top of its modelled grants (bought
  Boosts on a third of all turns).
- Two small supporting changes: `test_ai_vs_ai.js` gained a `--plan-ab[=N]` batch
  (planner vs legacy, seats alternated) alongside its existing `--focus` mode, and
  Baiken's/I-No's/Giovanna's character actions declare `alwaysStrikes`/`planCloses` in
  `cards.js` so the planner can model them without calling a `resolve()` that mutates
  real game state.
- New `test_ai_planner.js` — 39 assertions covering the invariants above plus a 25ms
  per-decision time budget (39 passed, 0 failed).
- **Measured**: `node test_ai_vs_ai.js --plan-ab=600` → planner 56% / legacy 44%,
  average life remaining 6.4 vs 4.3. Aggregate play quality also improved:
  out-of-strict-range Strikes 33.1% → 28.1%, average game length 22.3 → 32.2 turns
  (both sides play more patiently). Cost: `test_headless.js` 38s → 55s,
  `test_ai_difficulty.js` 201s → ~500s — the AI thinks harder on every turn, and both
  suites are AI-turn-bound.
- Verified: `node build.js` + `node -c bundle.js` clean; `test_ai_planner.js` 39/39;
  `test_guiltygear.js` 185/185; `test_blazblue.js` 110/110;
  `test_cammy_zangief_fix.js` 16/16; `test_headless.js` 0/200 crashes;
  `test_ai_vs_ai.js` 0/100 crashes; `test_ai_difficulty.js` 0 crashes, both ladder
  checks still `true` (Champion 86.7% ≥ Warrior 84.3% ≥ Student 73%).

## 2026-08-17 (later — Season 4 finished)

- **Every remaining Guilty Gear Strive fighter finished and made selectable**:
  chipp, ky, anji, may, ramlethal, nagoriyuki, baiken, faust, giovanna, ino,
  jacko, happychaos, goldlewis. Card data transcribed from `EXdata.xlsx` and
  Innate/Exceed/character actions wired for each — the same pass the BlazBlue
  fighters got. `zato1` is the only GG fighter still flagged unfinished (Eddie
  token system, and the workbook has no Zato-1 sheet at all).
- **The pre-existing card data for chipp/ky/anji/may/ramlethal was invented, not
  transcribed** — wrong card names, wrong stats, wrong printed text, and wrong
  Exceed costs for chipp/giovanna/ino — written in an earlier session that did
  not have the workbook. Replaced wholesale from the source sheets. See
  `HANDOFF.md` §4's Season 4 paragraph.
- **Two real bugs in older code, both invisible to the crash-counting suites**:
  `buildDeck`'s `c.copies || 2` turned a deliberate `copies: 0` back into 2, and
  `triggerOnCancelHooks` had no fallback from `onCancelExceed` to `onCancel` —
  which meant **Chipp's and Ky's Exceeds had never done anything**. Both fixed.
- New generic extension points rather than character branches (full list in
  `HANDOFF.md` §4b): per-card `liveStatMods` (one hook covering every
  conditional stat clause the GG cast needed), `dynamicRangeMaxBonus`,
  `hitImmuneToFasterAttacks`, `sealAfterUse`, `neverLeavesPlay`; per-character
  `invalidStrikeCard`, `cardCleanupFor`, `onReshuffle`, `canExceedAction` +
  `engine.forceExceed`, and `canGoFaceUp` as a per-card predicate.
  `player.revertPending` is now actually consumed, at the end of a Strike.
- Also removed the last `characterKey === '...'` branch left in `engine.js`
  (Nagoriyuki's Special invalidation), per §4b's own standing rule.
- New `test_guiltygear.js` — 185 assertions pinning every GG fighter's
  Innate/Exceed/action and the new primitives (185 passed, 0 failed).
- Verified: `node build.js` + `node -c bundle.js` clean; `test_guiltygear.js`
  185/185; `test_blazblue.js` 110/110; `test_cammy_zangief_fix.js` 16/16;
  `test_headless.js` 0/200 crashes; `test_ai_vs_ai.js` 0/100 crashes plus
  focused `nagoriyuki:goldlewis`, `happychaos:faust`, `baiken:ino`,
  `jacko:giovanna`, `chipp:ky` and `anji:may` runs at 0/60 crashes each;
  `test_ai_difficulty.js` 0 crashes, both ladder checks `true`.

## 2026-08-17

- **Four more BlazBlue fighters promoted to fully playable**: Iron Tager,
  ν-No. 13, Bang Shishigami, Hakumen. Their card data and most of the
  engine-side primitives were already transcribed/built by the Season 5 pass
  but had no `CHARACTERS` entry, so they were not selectable. This session
  wired the Innate/Exceed/Overdrive for each and fixed the gaps that only
  became reachable once they were:
  - `engine.spendGaugeIfAble(player, n)` — **did not exist**, yet four of
    Hakumen's cards already called it ("You must spend [1] if able"). Crashed
    4/200 games in `test_headless.js` the moment he became selectable. Same
    invented-API failure mode `HANDOFF.md` §7 documents for
    `doCharacterAction`/`pullToward` — worth grepping card data for engine
    methods that don't exist before promoting any future character.
  - `applySetTimeOption`'s Force branch called `pickExactForceCards(player,
    pool, cost)` — that method takes `(player, cost)`. Silently made every
    Force-cost set-time option unpayable; Nu-13's Innate is the first one, so
    nothing had exercised it before. The CPU's own accept heuristic in
    `ui.js` had the matching bug (it checked the Gauge count for a Force
    cost), fixed to check whichever resource the option actually costs.
  - `pushToward`/`pullTowardWithoutPassing` implement a Pull as an *advance of
    the target*, so they were running the target's own self-movement
    restrictions — Iron Tager's cap and Jin's/Nu-13's "the opponent cannot
    move" — against being Pulled, which both cards' text (and
    `cappedSelfMove`'s own comment) says must not happen. `advanceSelf` takes
    a `forced` flag now; both Pull helpers pass it.
  - Two flags the Season 5 card data declared but nothing read:
    `spdPerGaugeCard` (Spark Bolt's "+1 SPD per card in Gauge", now computed
    in `liveStats`) and the `movementImmuneGrant` boost flag ("You cannot be
    Pushed or Pulled", now checked by a shared `isPushPullImmune`).
  - New `engine.shuffleDiscardIntoDeck(player)`, split out of
    `manualReshuffle` for Hakumen's Exceed — same shuffle, but it must not
    consume the once-per-game Reshuffle use or fire the Astral Heat trigger.
- **Platinum the Trinity deliberately left unwired** despite her card data
  being complete — her Innate/Exceed both hang off a Cleanup step that has no
  character-level hook, and would have to call `playBoost` from inside cleanup
  (Force cost + `triggersStrike` follow-up). Documented in `HANDOFF.md` §4
  rather than half-implemented.
- `test_blazblue.js` extended from 48 to 110 assertions covering all four new
  fighters (48 → 110, 0 failed); its shared-mechanics loop now runs over all
  eight implemented Season 5 fighters.
- **Fighter photos for Chun-Li, M. Bison, Millia and Giovanna** — 13 of the
  roster now have one. Applied the lessons in `render.js`'s `FIGHTER_PHOTOS`
  comment up front instead of rediscovering them: all four source files
  checked for the bottom white-strip defect first (none had it); JPEG at
  quality 88 rather than PNG (the earlier PNG batch was a one-off user request
  and costs ~5-10x the bytes for photographic content); three resized to the
  usual 480px wide, while `mbison2.jpg` arrived at 452px wide and was embedded
  untouched — upscaling would invent pixels and a JPEG→JPEG re-encode would
  only add generation loss. Verified by extracting all 13 data URIs back out
  of the built `exceed_poc.html` and decoding them: correct format and
  expected pixel dimensions for each.
- Verified: `node build.js` + `node -c bundle.js` clean; `test_blazblue.js`
  110/110; `test_cammy_zangief_fix.js` 16/16; `test_headless.js` 0/200
  crashes (was 4/200 before the `spendGaugeIfAble` fix); `test_ai_vs_ai.js`
  0/100 crashes plus focused `tager:hakumen` and `nu13:bang` runs at 0/60
  crashes each; `test_ai_difficulty.js` 0 crashes, both ladder checks `true`.

## 2026-08-16 (continued — real-browser playtest)

- **First full real-Chrome playtest of the built `exceed_poc.html`** (Playwright
  driving installed Chrome, no browser download; harness lived in a scratch
  directory, see `HANDOFF.md` §1 for how to rebuild it). Drove ~30 complete
  games through the real DOM and reached, screenshotted, and visually checked
  every interactive mode added in this session's Tier 1-4 work:
  `face_up_choice` (Sagat, both the attacker prompt and the defender's
  "set face up: <card>" reveal), `named_choice_effect` (Millia — correctly
  offers 3 options + Decline while un-Exceeded), `may_exceed_choice` (Leo),
  `action_gauge_select` and `char_exceed_action_choice` (M. Bison),
  `retreat_after_boost_choice` (Chun-Li), plus the core loop (strike/defend/
  critical/grasp/boost/change-cards/move/strike-result). **All render
  correctly**; zero page errors, zero console errors, no literal
  `undefined`/`NaN`/`[object Object]` in any rendered output, no off-screen or
  zero-size controls.
- **Fixed the one real defect the playtest found**: `#app` was capped at
  `max-width:900px` at every viewport size, but a full 7-card hand is ~1090px
  of card tiles — so the last two cards sat outside `.hand-row` and could only
  be reached by scrolling it sideways, even on a 1920px screen with a third of
  the window empty. Added a single media query (`@media (min-width:1220px){
  #app{max-width:1180px} }`) in `shell.html`; measured hand-row overflow at
  1920/1400px drops from 216px to 0, and the measured layout at 1024px/390px is
  unchanged.
- Confirmed *not* bugs while investigating: the character-select blurbs that
  end in "…" are deliberately line-clamped (no card's content overflows its
  tile — all 32 measure clean at a uniform 300px); and `openHowToPlay()`
  throwing after a game starts is only reachable from a test harness, since
  the button and the overlay both live inside `#app` and in-game there is no
  How to Play button (see `HANDOFF.md` §1).
- Verified after the CSS change: `node build.js` clean, `node -c bundle.js`
  clean, `test_cammy_zangief_fix.js` 16/16, `test_headless.js` 0/200 crashes,
  `test_ai_vs_ai.js` 0/100 crashes, and the browser layout pass re-run at all
  four viewport sizes.

## 2026-08-16 (continued — Tier 3)

- **New engine primitive: "Normals gain an on-Hit effect."** New
  `CHARACTERS[key].normalsHitHook` generator, dispatched via `yield*` in
  `engine.js`'s `activateStep` (gated on `card.type === 'normal'`, mirrors
  `normalsStatMods`/`normalsRangeBonus`'s existing gating pattern). Unblocks
  **Testament** (Innate/Exceed: "Your Normals have 'Hit: [Draw 1.] You may
  Push or Pull 1.'" — reuses the existing `grasp_move` yield with `min:0` to
  allow declining, zero new UI needed). **Promoted Testament to fully
  playable.**
- **New engine primitive: range-conditional stat bonus on ALL attacks, not
  just Criticals.** New `CHARACTERS[key].rangeConditionalStatBonus(player)`
  in `engine.js`'s `liveStats` (mirrors the existing Critical-only
  `extraCriticalStatsIfOpponentAtRange1`, applied unconditionally instead).
  Unblocks **Potemkin** (Innate/Exceed: "+1/+2POW and +1/+2ARM when
  initiating at Range 1"). **Promoted Potemkin to fully playable.**
- **New engine primitive: generic "pick one of N named effects" yield.** New
  `named_choice_effect` yield type + UI mode (options list + optional
  Decline button, `ui.js`/`render.js`), plus a `may_exceed_choice` yes/no
  yield for the narrower binary case. Both dispatch through
  `CHARACTERS[key].onAfterStrike`, converted from a plain function call to a
  `yield*`-driven generator call in `engine.js` (fires after every Strike) —
  this is what actually unblocked both of the below, not just the new yield
  type on its own.
  - **Millia**: Innate/Exceed "may draw 1, Advance 2, Retreat 2, or gain
    Advantage" — previously silently simplified to always Draw 1 with no
    code backing that claim at all (docs were aspirational, not real).
    **Promoted Millia to fully playable.**
  - **Leo**: "may Exceed if opponent stunned" was previously an unconditional
    auto-Exceed (a real simplification the docs correctly flagged, in §7).
    Now a genuine yes/no prompt. Leo was already in the "fully playable" list
    — this is a quality fix, not a promotion.
  - AI heuristics added for both new yield types
    (`resolveAIStrikeRequest` in ai.js) and `test_headless.js` driver cases
    added for both new `G.mode` values (HANDOFF.md §5's standing rule).
- **Giovanna's "Hit: Draw N" implemented** via a sibling primitive,
  `CHARACTERS[key].actionStrikeHitHook` (narrower than `normalsHitHook`:
  gated on the pre-existing `actionStrikeThisTurn` flag, so it applies only
  to the one Strike her character action triggers, not every Normal) — not
  promoted to fully playable (her Exceed action's asymmetric [1]-only-when-
  Exceeded Force cost is still uncharged, see HANDOFF.md §6).
- Corrected three stale/aspirational doc claims found while implementing the
  above: Millia's "simplified to Draw 1" had no backing code at all; ky was
  audited against chipp/anji/may/ramlethal and found already fully working
  (see Tier 0 above); `HANDOFF.md` §4/§6/§7 rewritten throughout this tier to
  match reality.
- Verified after each sub-change and again at the end of the tier: `node -c
  bundle.js` clean; `test_headless.js` 0/200 crashes (exercises the new
  interactive yields via the full roster, including the newly-promoted
  characters); `test_ai_vs_ai.js` 0/100 crashes; `test_cammy_zangief_fix.js`
  16/16.

## 2026-08-16 (continued — Tier 5, 2-ply lookahead decision)

- **Considered building 2-ply AI lookahead, deliberately deferred it again.**
  This file's own prior analysis (`HANDOFF.md` §6) already argued ply-1
  fidelity should be fixed first, since a deeper search over an already-noisy
  per-ply evaluator amplifies noise rather than averaging it out — re-checked
  that reasoning and it holds. It's also a large, performance-sensitive,
  hard-to-validate-quickly change (nested Monte Carlo sampling) with no fast
  way to confirm a net improvement outside a long human-vs-AI playtesting
  session. Explicit, considered deferral, recorded as such rather than left
  looking like an oversight.
- While investigating, found "no live opponent Gauge tracking" (listed
  alongside 2-ply as a ply-1 fidelity gap) is narrower than it sounds: the
  opponent's Gauge is already full public information
  (`publicInfoOf().gauge`, exact card keys) and already used for one thing
  (the AI's own Ultra-affordability check, own side only). What's actually
  missing is a scoring-heuristic addition (reasoning about whether the
  OPPONENT can afford to answer with a Critical/Ultra/EX), not new plumbing
  — corrected `HANDOFF.md`'s framing. Also fixed a stale duplicate bullet
  there ("Boost's force-payment picker only offers hand") that had already
  been fixed in this session's Tier 1 work but never removed from the gaps
  list.
- No code changes this round (documentation/decision-recording only).

## 2026-08-16 (continued — Tier 5, full AI/roster mechanic-coverage audit)

- **Audited every "fully playable" character not yet covered by the
  2026-08-14 Ryu/Ken/Zangief/Cammy/Vega/Akuma/C.Viper pass**, per HANDOFF.md
  §6's own standing follow-up request. Found three characters silently
  shipping a completely non-functional Innate/Exceed (zero code outside
  cards.js despite being listed "fully playable" — a real, player-facing gap,
  not just a documentation nuance) and fixed all three:
  - **M. Bison**: "Action: Add [up to 3] card(s) from hand to Gauge, draw
    that many; Exceed may take this immediately." New
    `needsHandToGaugeChoice` action shape + dedicated `action_gauge_select`
    UI mode (the generic `action.resolve` signature can't express a
    multi-card pick), `char_exceed_action_choice` mode for the "may act
    immediately on Exceed" follow-up (mirrors the existing `mayStrike`
    shape), AI heuristic (`aiRunHandToGaugeAction`) + `scoreCharacterAction`
    branch.
  - Along the way, fixed a related bug: the AI's `runAITurn` `'exceed'`
    branch discarded `doExceed`'s return value entirely, so Anji/May's own
    Exceed-time `mayStrike` bonus was never followed up by the AI (only the
    human side handled it). Both `mayStrike` and the new `mayAction` are now
    honored on the AI side.
  - **Chun-Li**: "Innate — After you resolve a Boost, you may Retreat 1" was
    a commented-out dead field with a comment claiming a yield existed that
    didn't. New `retreat_after_boost_choice` yield, inserted into `doBoost`'s
    continuation chain before any follow-up Strike (per her own printed "If
    you would Strike, retreat first" clarification).
  - **Dan**: "Innate — Your Wild Swings have 'Critical: +1POW and +1SPD'".
    New `wildSwingCriticalStatMods` extension point (gated on the
    pre-existing `fromWildSwing` param, Leo's `wildSwingStatMods` precedent)
    + threaded `fromWildSwing` through the AI's Critical-payment decision
    chain (`hasRealCriticalValue`/`criticalPayProbability`/
    `chooseCriticalPayment`/`aiDecideCritical`, previously dist-only) so the
    AI values paying Critical on its own Wild Swing correctly. His Exceed's
    "draw from top or bottom of deck" is a real, separate, still-open gap
    (a comment claiming it was already handled was aspirational) — left
    honestly documented, not silently claimed fixed.
  - **Guile**: also completely unimplemented, but genuinely ambiguous
    without the source rulebook (no existing mechanism for Change Cards to
    take an optional Gauge payment, and Force isn't a banked resource
    anywhere in this engine). Did not guess at an implementation — moved
    from "fully playable" to `shell.html`'s unfinished list instead, so the
    game stops silently offering a non-functional fighter.
  - `HANDOFF.md` §4 rewritten with full detail on all four; none of the four
    needed new card data (unlike the Tier 4 audit's Nagoriyuki/Happy
    Chaos/Goldlewis), so three were fixable outright and the fourth honestly
    flagged.
- Verified after each character's fix and again at the end of the tier:
  `node -c bundle.js` clean; `test_headless.js` 0/200 crashes; `test_ai_vs_ai.js`
  0/100 crashes; `test_cammy_zangief_fix.js` 16/16.

## 2026-08-16 (continued — Tier 4, Nagoriyuki/Happy Chaos/Zato-1/Goldlewis audit)

- **Audited the remaining blocked-character claims in HANDOFF.md §7** and
  found the "Revert (un-Exceeding)" primitive gap was stale — `engine.
  doRevert(player)` already exists and is genuinely wired (used by Leo at
  turn-end already). Its own header comment's worry about "Exceed-Mode-only
  cards" turned out to be moot: this engine has no such card concept at all
  (confirmed via grep). **Nagoriyuki's real remaining blocker is missing
  source data**, not missing plumbing — his `engine.isValidStrikeCard`
  (Special-invalidation-while-Exceeded), `engine.doRevert`, and `engine.
  takeDamageNonLethal` primitives are all already real and wired; only his
  `specials: []` (needs `EXdata.xlsx`'s "Nagoriyuki" sheet, not present in
  this environment) blocks him. Did not fabricate his card data — per this
  project's own standing rule, card text must be transcribed from the
  workbook, never invented.
  - **Happy Chaos**: same pattern — the two primitives its own header comment
    called BLOCKED (`engine.sealCardFromHand`, `engine.doRevert`) already
    exist generically. Real remaining blockers: his `specials: []` (same
    missing-source-data issue) and a still-unbuilt interactive yield for his
    Innate's "Hit: may seal an Ultra" choice.
  - **Goldlewis**: audited and found already-accurate — `maxHandSize: 9` and
    `onStartingHand` (2 extra starting-hand cards) are both genuinely wired
    (`onStartingHand` call site is in `ui.js`'s `GameController` constructor,
    not `engine.js`, which is why an engine.js-only grep during an earlier
    audit missed it). Only `specials: []` blocks him — same missing-source-
    data issue as Nagoriyuki/Happy Chaos.
  - **Zato-1**: confirmed his blocker (the Eddie second-controllable-position
    system) is genuinely large, novel scope with no existing primitive to
    reuse — the one gap in this batch that was already accurately described.
  - `HANDOFF.md` §4/§7 and the corresponding `cards.js` header comments
    rewritten to match; none of these four were promoted (all still lack
    real card data or, for Zato-1, a large unbuilt system).
- Verified: `node -c bundle.js` clean; `test_headless.js` 0/200 crashes
  (doc/comment-only changes this round, no behavior change).

## 2026-08-16 (continued — Tier 4, Sagat)

- **Implemented Sagat's face-up Strike mechanic** (the biggest single item in
  the plan so far — HANDOFF.md previously called this "the HARD PART" and
  deferred it). New `CHARACTERS[key].canGoFaceUp`/`.faceUpStatBonus(player)`
  + a real strike-time choice, offered right after picking the strike card
  and before the Critical prompt: new `face_up_choice` UI mode
  (human, via `doHumanStrike`) and a new `chooseFaceUp` AI heuristic (AI, via
  `aiInitiateStrike`). Sets a new per-strike flag
  (`player._faceUpThisStrike`, same pattern as the existing
  `_unhittableThisStrike`/`_rangeImmuneThisStrike`), read by `liveStats` for
  the stat bonus. Skipped on a Wild Swing (matches `maybeAskCritical`'s
  existing `hideCardName` reasoning — even the owner doesn't know that card
  yet). When a face-up reveal happens, `defend_select` now shows the real
  card name to the human defender. Also fixed Sagat's Exceed "Retreat up to
  2" (was a no-op TODO) to auto-resolve to the max, this codebase's
  established "up to X" convention. **Promoted Sagat to fully playable.**
  Documented scope limit in HANDOFF.md §7: the AI's own defense heuristic
  doesn't yet specially exploit a human's face-up reveal (still samples
  blind) — a real, deliberate, documented simplification, not a bug.
- Verified: `node -c bundle.js` clean; `test_headless.js` 0/200 crashes
  (exercises the new `face_up_choice` mode via the full roster);
  `test_ai_vs_ai.js` 0/100 crashes; `test_cammy_zangief_fix.js` 16/16.

## 2026-08-16 (continued — Tier 2)

- **Audited the three "evalMatchup doesn't model X" gaps HANDOFF.md §6 had
  listed** and found two were stale (already fixed as a side effect of the
  2026-08-14 Akuma/Vega/C.Viper Critical-modeling work, never reflected in
  the docs): Criticals ARE modeled generically for every card via
  `criticalGrant()`/`criticalPayProbability()`, and Zangief's Critical-gated
  immunities (Banishing Flat, Flying Power Bomb) use that exact same generic
  path. Also confirmed Zangief's pass-through blocking is real but has no
  measurable effect on `evalMatchup`'s scoring given its 1-D
  distance-only model (see HANDOFF.md §6 for the detail). Cammy's
  sustain/boost-expiry timing gap is real but judged too speculative to fix
  safely right now — left open, documented. `HANDOFF.md` §6 rewritten to
  match reality instead of re-doing already-done work.
- **`scoreCharacterAction` (ai.js) extended beyond ryu/ken** — new scoring
  branches for baiken, ino, jacko, giovanna, faust (all mechanically
  functional today even though UI-flagged unfinished for other reasons).
  Gated on `actionForceCost` affordability where relevant so the AI doesn't
  pick an action it can't pay for.
- Verified: `node -c bundle.js` clean; `test_headless.js` 0/200 crashes;
  `test_ai_vs_ai.js` 0/100 crashes.

## 2026-08-16 (continued — Tier 1)

- **Boost force-payment picker now accepts Gauge, not just hand** (`ui.js`'s
  `toggleForceCard`/`render.js`'s `boost_force_select` block) — engine-side
  `playBoost` already supported hand-or-gauge via `payForceFromHandOrGauge`;
  only the human UI picker and its render were hand-only. Mirrors Move's
  existing `move_force_select` pattern. AI side already picked from hand+gauge
  via `pickForceUidsForTarget`, unaffected.
- **AI now uses manual Reshuffle.** New branch in `ai.js`'s `chooseAction()`:
  triggers when the AI's own deck has ≤1 card left, Reshuffle hasn't been used
  yet this game, and the discard pile has ≥3 cards worth reclaiming — checked
  after the lethal-strike check so it never trades away a kill. New
  `ui.js` `runAITurn()` branch calls `engine.manualReshuffle(player)`.
- **Fixed Dive/GG Dive's "If side swap, cannot be hit" clause** — previously
  set `ctx.sideSwapped` but nothing read it to block the opponent's attack.
  Added `player._unhittableThisStrike` (mirrors the existing
  `_rangeImmuneThisStrike`/`_focusActive` per-strike flag pattern): Dive's
  `before` hook sets it on the attacker when the side-swap happens; the
  hit-check in `engine.js`'s `resolveStrikeSteps` now reads it off the
  DEFENDER before deciding a hit connects. Reset in the same cleanup block as
  the other per-strike flags.
- **Baiken/I-No/Jack-O's character actions now really cost [1] Force**, as
  printed (previously simplified to free, per `HANDOFF.md` §6). New
  `CHARACTERS[key].actionForceCost` field (data-driven, not per-character
  branching) + a new paid-action UI flow: `ui.js`'s `runCharacterAction` now
  pauses at a new `action_force_select` mode (hand-or-gauge picker, same
  shape as Boost/Move's) before actually resolving the action; AI side pays
  via the existing `pickForceUidsForTarget` before resolving, and skips the
  action for the turn if it can't pay exactly. `test_headless.js` got a
  matching driver case for the new mode (HANDOFF.md §5's standing rule).
- **Jack-O's Exceed "draw 3" wired** — new `onExceed` hook
  (`engine.draw(player, 3)`), closing the last item in that TODO.
- Updated the corresponding `UNFINISHED_CHARACTER_NOTES` entries in
  `shell.html` for baiken/ino/jacko to stop claiming the Force costs are
  simplified/free.
- Verified: `node -c bundle.js` clean; `test_ai_vs_ai.js` 0/100 crashes;
  `test_headless.js` 0/200 crashes (roster includes unfinished characters, so
  this exercises the new paid-action flow for baiken/ino/jacko too);
  `test_cammy_zangief_fix.js` 16/16; `test_ai_difficulty.js` ladder trend
  held.

## 2026-08-16

- Added `build.js`, a Node reimplementation of the python3 bundling step
  (python3 isn't available in this environment; see `HANDOFF.md` §2/§1 for
  why one was needed) — run with `node build.js` from `exceed/`, replaces the
  inline python3 heredocs.
- **Promoted `ky` to fully playable.** Audited it against the other four
  characters flagged unfinished-but-maybe-not (chipp/anji/may/ramlethal, per
  `HANDOFF.md` §6): ky's Innate/Exceed ("first Cancel each turn: [draw 1,]
  may Advance/Retreat 1") is fully wired end-to-end — `onCancel` in
  cards.js, `triggerOnCancelHooks` in engine.js, and a real
  `move_direction_choice` UI prompt in ui.js (3 call sites). Chipp, anji,
  may, and ramlethal were checked too and confirmed to still have real
  unimplemented core mechanics (zero matches for their character key in
  engine.js beyond `onExceed`) — left unfinished, their notes corrected to
  stop implying they're closer to done than they are (ramlethal's old note
  said "should be fully playable", which wasn't true).
- **Consolidated the 5 duplicate unfinished-character-key lists in
  `shell.html`** (`HANDOFF.md` §6 tech-debt item) into one
  `UNFINISHED_CHARACTER_NOTES` object + derived `UNFINISHED_CHARACTER_KEYS`
  array, referenced by `fighterCardHTML`, `renderCharSelect`,
  `renderAiCharSelect`, `selectCharacter`, `selectAiCharacter`. Only one
  place to edit now when a character's status changes.
- Verified via `node build.js && node -c bundle.js`, `test_ai_vs_ai.js`
  (0/100 crashes), `test_headless.js` (0/200 crashes), `test_ai_difficulty.js`
  (0 crashes, ladder trend holds) — all four run once as a baseline before
  any change this session, then re-run clean after the above.

## 2026-08-15

- **Replaced Akuma and Sol Badguy's abstract SVG pictogram silhouettes with
  real photos too** (`akuma.png`/`sol.jpg`), same `FIGHTER_PHOTOS` mechanism/
  480px-wide resize pipeline as the Axl/C. Viper/Vega entry just below — see
  `HANDOFF.md` §6 for current status (9 of 14 fully-playable fighters now
  covered). Both source files were already sitting in the project folder
  (akuma.png 502×860 RGBA 811KB; sol.jfif — a JPEG despite the extension —
  711×1008 186KB). Differences from the axl/cviper/vega batch: kept each in
  its existing format rather than force-converting to PNG (Akuma was already
  PNG; Sol was already JPEG, renamed `sol.jfif` → `sol.jpg`, re-encoded at
  quality 88 during the resize — JPEG is the legitimately better fit for a
  photographic image, no reason to convert it to PNG just for consistency).
  Akuma's source PNG had an alpha channel that turned out to be unused —
  sampled ~900 pixels across the image and confirmed 100% opaque before
  dropping it to 24bpp RGB during the resize (a real, verified size win, not
  an assumption). Checked both for the white-strip defect — neither had it.
  Verified the same way as the axl/cviper/vega batch: DOM inspection in a
  real browser (character-select cards + a live Akuma-vs-Sol-Badguy game's
  life panels), correct `data:image/png`/`data:image/jpeg` src and correct
  480px-wide decoded dimensions, full test suite still 0 crashes.
- **Replaced Axl Low, C. Viper, and Vega's abstract SVG pictogram silhouettes
  with real photos** (`axl.png`/`cviper.png`/`vega.png`), extending the
  `FIGHTER_PHOTOS` mechanism built for Cammy/Ryu/Ken/Zangief to three more
  fighters — now 7 of 14 fully-playable characters have one (see `HANDOFF.md`
  §6). Source files were already sitting in the project folder at ~2.2–2.5MB
  each (1024×1536/1122×1402, truecolor PNG) — resized down to 480px wide (the
  largest this project ever actually renders a fighter photo at —
  `.fighter-card{width:230px}` in shell.html, doubled for high-DPI headroom)
  before embedding, cutting combined size from ~6.5MB to ~2.3MB while keeping
  full lossless color. Tried real palette quantization first (8bpp indexed
  PNG via GDI+) for a bigger size win, rejected it — GDI+'s default indexed
  conversion has no proper dithering and visibly posterized photographic
  content (banding/color blocks); resize-only was the right trade-off given
  no other image-compression tooling (pngquant/ImageMagick/sharp/etc.) was
  available in this environment. Checked all three for the white-strip defect
  documented in `render.js`'s `FIGHTER_PHOTOS` comment (present on all four
  earlier photos) — none of the three new ones had it. Kept PNG format per
  explicit user request, unlike the earlier four which are JPEG. Verified via
  DOM inspection in a real browser session (same throwaway local-server
  approach as the horizontal-life-panels session): correct `data:image/png`
  src on the character-select cards and in a live game's life panels
  (Axl-vs-C.-Viper), images decode at the expected 480px-wide dimensions, full
  test suite (`test_headless.js`/`test_ai_vs_ai.js`) still 0 crashes — this
  only touched `render.js`'s data map, no game logic.

## 2026-08-14

- **Consolidated `test_cammy_zangief_games.js` into `test_ai_vs_ai.js`,
  deleted the former.** Per user request ("consolidate, cull duplicity") after
  discussing whether the two Cammy/Zangief-specific test files were redundant
  — `test_cammy_zangief_fix.js` (16 unit assertions pinning exact numeric
  behavior) was judged genuinely non-redundant and kept as-is;
  `test_cammy_zangief_games.js` (a second, hand-rolled AI-vs-AI turn loop that
  duplicated `test_ai_vs_ai.js`'s `playGame()` just to add per-focus-character
  stat tracking) was the real duplication, already flagged as tech debt in
  `HANDOFF.md` §6. Folded the tracking directly into `test_ai_vs_ai.js`'s
  existing `playGame`/`doStrike` (always computed now, cheap) and exposed it
  via a new opt-in `--focus` CLI flag (`node test_ai_vs_ai.js --focus` runs
  the same canned Zangief/Cammy/Ryu-mirror comparison the old file did;
  `--focus=charKey:oppKey` runs any other pair, e.g. `--focus=vega:ryu`,
  generalizing what used to be hardcoded to exactly two characters). Default
  `node test_ai_vs_ai.js` (no flag) behavior/output is byte-for-byte
  unchanged — verified by diffing its aggregate-stats output before and after.
  Caught and fixed one real bug in my own new code before shipping it: the
  initial gate used `process.argv.includes('--focus')`, which is exact-match
  and never matches `--focus=vega:ryu` (a different string) — silently made
  the `--focus=...` form a complete no-op with no error. Fixed to check
  `a === '--focus' || a.startsWith('--focus=')`. Verified all three modes
  (default, bare `--focus`, `--focus=vega:ryu`) produce sensible 0-crash
  output, plus a full re-run of `test_headless.js`/`test_cammy_zangief_fix.js`
  to confirm no regression from touching the shared `playGame`/`doStrike`.
  `HANDOFF.md` §2/§5/§6 updated to match (file manifest, test-suite docs, and
  removing the now-resolved tech-debt bullet).
- **Fixed all three CPU-AI gaps found in the C. Viper/Vega/Akuma audit** (see
  the audit entry below and `HANDOFF.md` §6 for full per-character detail).
  Used three parallel subagents, each in an isolated git worktree (to avoid
  clobbering each other's edits to shared files, `ai.js` especially). Summary:
  - **Vega**: `evalMatchup` now values his edge-position POW bonus;
    `bestMoveTarget` nudges toward an edge when his hand favors Specials/Ultras.
  - **Akuma**: implemented the actually-missing mutual-Critical-POW engine
    mechanic (`CHARACTERS.akuma.grantsMutualCriticalPow`, applied to both
    sides' `liveStats` in `engine.js`), then wired `ai.js` to recognize it.
  - **C. Viper**: `evalMatchup`/`estimateBoostTimingValue` now understand her
    Ultra's continuous-boost requirement, her innate Stun Immunity, and her
    conditional POW bonus/opponent debuff.
  - All three verified independently by their agents: full test suite (0
    crashes, ladder holds) plus a standalone `node` script proving the actual
    stat delta.
  - **Merging the three worktrees back was not a clean copy** — two real
    problems surfaced and were fixed during the merge itself, worth recording
    as a lesson: (1) all three agents' worktrees had branched from the last
    **git commit**, not this session's uncommitted working-tree changes —
    they were missing every prior fix from today (Prepare AI, Wild Swing note,
    loud-discard/draw, horizontal panels, the earlier `doCharacterAction` fix,
    the compressed `HANDOFF.md`/`CHANGELOG.md`). Each agent independently
    re-discovered and re-fixed the `doCharacterAction` bug against their own
    stale base — harmless (identical fix) but confirms the pattern. Handled by
    manually reviewing each worktree's `git diff` and hand-porting just the
    substantive `ai.js`/`engine.js`/`cards.js` hunks onto the current files,
    not copying any file wholesale. (2) In doing that by hand, the `cards.js`
    half of the Akuma fix (`grantsMutualCriticalPow` itself) was reviewed but
    never actually applied — an oversight in the merge process, not the
    subagent's work. It went undetected by the full test suite (0 crashes
    either way, since the missing field just made the new code paths silent
    no-ops) and was only caught by re-running the same standalone verification
    script the agent had used, which showed no stat delta. **Lesson: a green
    test suite proves the code didn't crash, not that a merge was complete —
    always re-run the standalone verification script for what you just merged,
    not just the crash-counting test suite.** If future sessions use worktree
    isolation for multiple parallel agents touching the same files again,
    consider having each agent commit its own worktree branch on top of the
    *current* HEAD (or rebase before reporting back) rather than branching
    once from stale history, to make merging closer to a real 3-way merge.
- **Audited whether the CPU AI plays C. Viper, Vega, and Akuma intelligently**
  (read-only, no code changes) — same methodology as the earlier Session 7
  Zangief/Cammy audit, run via three parallel subagents. All three came back
  with real gaps, now logged in `HANDOFF.md` §6: Vega's entire Innate/Exceed
  edge-position mechanic is invisible to `evalMatchup`/movement logic; Akuma's
  headline "mutual Critical bonus" mechanic isn't implemented in the engine
  at all (not just an AI gap); C. Viper's Continuous-Boost-engine gimmick is
  real but never credited by `evalMatchup`/`scoreBoostCard`/
  `estimateBoostTimingValue`, and her Ultra can be picked with zero chance to
  connect. No fixes applied yet — this was audit-only, per the user's request.
- **Fixed a real, pre-existing crash**: `ui.js`'s `runAITurn()` called
  `this.engine.doCharacterAction(...)`, a method that never existed — crashed
  roughly 1 in 50 games (4/200 in `test_headless.js`), invisible until Node
  became available again to actually run the suite. Fixed to match the
  correct, already-working human-side call pattern
  (`CHARACTERS[key].action.resolve(engine, player, opponent, direction)`).
  See `HANDOFF.md` §7 pitfalls (the invented-API list).
- Rebuilt the project via Node for the first time in a while (Node had been
  unavailable for several prior sessions) and ran the full test suite for
  real: `test_ai_vs_ai.js` 0/100 crashes, `test_headless.js` 0/200 crashes,
  `test_ai_difficulty.js` 0 crashes with the ladder holding,
  `test_cammy_zangief_fix.js` 16/16, `test_cammy_zangief_games.js` 0 crashes.
- **AI**: `chooseAction()`'s Prepare fallback is now scored instead of blind —
  new `preferChangeCardsOverPrepare(hand, dist, boostMods)` in `ai.js` makes
  the AI prefer ChangeCards over Prepare when the hand has significant dead
  weight (non-boost cards currently out of range), instead of always just
  drawing a card on top of unusable ones.
- **UI**: the `defend_select` panel now notes when the CPU's Strike is a Wild
  Swing ("CPU struck (Wild Swing)! ...") — the card's identity stays hidden,
  only the fact that it was a Wild Swing (already public information
  elsewhere in the game, e.g. Focus's boost) is surfaced.
- **Logging**: a Stunned card's forced discard now names the card
  (`resolveStrikeSteps`) instead of just saying "their attack is discarded" —
  closes the one path that was still violating the "loud discard" invariant.
  `engine.draw()` now logs the human ("You") player's drawn card names for
  every draw path (Prepare, Mulligan, boost-granted draws, opening hand,
  etc.) — deliberately not done for the CPU, which would leak its hand. Both
  are now standing invariants, see `HANDOFF.md` §7 (#4 and #4b).
- **UI**: the life panels are now horizontal — You on the left, CPU on the
  right — instead of stacked vertically with CPU on top. Verified in a real
  browser session (not just headless/Node) via a throwaway local static
  server and direct DOM measurement: correct left/right order, no horizontal
  overflow at mobile (375px) or desktop width, correct rendering with the
  EXCEEDED badge and an active boost both present.
- `HANDOFF.md` compressed from ~3060 lines to ~550 — deduplicated repeated
  "READ THIS FIRST" session banners and narrative retelling into a flat,
  current-state reference (architecture, roster status, known gaps,
  invariants, pitfalls), while preserving every concrete fact. This is why
  this changelog can't give exact dates for most work before today: the
  session-by-session dating that used to live in `HANDOFF.md` was
  deduplicated away in that pass, and only two dated markers ("Session 7" and
  "Session 6", both this same day; "Session 3", the day before) were still
  remembered at the time this changelog was written. See "Earlier work"
  below.
- `CLAUDE.md` created at the repo root (harness guidance: build/test commands,
  architecture summary, invariants) — not part of the shipped game, but
  recorded here since it's a repo-level artifact from this stretch of work.
- This `CHANGELOG.md` itself created, at explicit user request — per user
  instruction, all present and future changes get logged here, dated, going
  forward. `HANDOFF.md` now points to it as the standing rule (added at its
  top).

## 2026-08-13 and earlier — "Session 3" through the original POC

Real work happened across several earlier sessions, but exact per-session
dates beyond "2026-08-13" (confirmed for the session that added Testament/
Potemkin/Leo Whitefang/Millia Rage, the 4th Guilty-Gear-rollout batch) and
"2026-08-14" (confirmed for the two sessions that immediately preceded
today's, which fixed a critical `actionSpecial()` test-coverage gap and did
an AI mechanic-coverage audit) are not recoverable from this repo — there is
only one git commit ("Initial snapshot", 2026-08-14) covering the entire
prior history flattened together, and `HANDOFF.md`'s own chronological
framing was compressed away today (see above) before this changelog existed
to capture it. If precise historical dates ever matter, they are gone; what
survives is the aggregate current-state description in `HANDOFF.md`
(architecture, full character roster and its status, known gaps, invariants,
pitfalls) — treat that as authoritative for "what's true now," and this
changelog as authoritative for "what changed" only from today (2026-08-14)
onward.

Known highlights from that earlier history, for context (see `HANDOFF.md` for
current status of each — some of this was later revised):

- Original proof-of-concept: Ryu, Ken, Cammy, Zangief (Street Fighter cast)
  plus Axl Low (first Guilty Gear character, own Normals pool). Core systems:
  turn loop, strike resolution, EX Strikes, Grasp/Block/Focus as real
  interactive decisions, a 3-tier AI difficulty ladder, Mulligan, Criticals,
  the Cancel mechanic, fighter photos, How to Play modal.
- A critical test-coverage gap was found and fixed: `test_headless.js`'s
  random action-picker never called `actionSpecial()`, meaning every
  character-action code path had a 0% chance of executing in any automated
  test for as long as the pattern existed. Fixing it immediately surfaced
  several invented-API bugs (`engine.pullToward`, `engine.movePlayer`,
  `this.gameId` — none ever existed) and an 11-character missing-`normals:`
  data bug.
  Fixing the AI's own `chooseAction()` to reach the same character-action
  path came later (Ryu/Ken only — see `HANDOFF.md` §6 for why other
  action-having characters still don't get it) and found a further
  range-legality bug affecting the whole roster's Before-mover cards
  (hit hardest: Zangief), plus three more bugs specific to Cammy.
  Testament, Potemkin, Leo Whitefang, and Millia Rage were added as part of
  an 18-character Guilty Gear rollout plan (Phase 1 of 5) — full detail on
  what's blocked for each is in `HANDOFF.md` §4/§7.
