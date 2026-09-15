# Exceed POC — Handoff Document

## Changelog

`CHANGELOG.md` (same directory) is the dated, factual log of what changed and
when. **Every session that changes this project must add a dated entry there
before considering the change done** — standing rule, added 2026-08-14. This
file (`HANDOFF.md`) stays organized by topic (architecture, current status,
gaps, invariants, pitfalls) rather than chronologically — don't try to
reconstruct history from it; that's what the changelog is for.

## Build/test status as of the most recent session

Five source changes accumulated across a stretch of sessions with no Node.js/
Python available (documented individually where relevant below: `ai.js`'s
`preferChangeCardsOverPrepare` Prepare-vs-ChangeCards heuristic; `render.js`'s
Wild Swing note on the defend prompt; `engine.js`'s Stun-discard and draw-log
"loud discard"/"loud draw" fixes; `shell.html`/`render.js`'s horizontal You-left/
CPU-right life panels). Node became available and all of them were finally
rebuilt and run through the real toolchain in one pass. That rebuild **also
surfaced and fixed one real, pre-existing, unrelated crash**: `ui.js`'s
`runAITurn()` called `this.engine.doCharacterAction(player, opp,
action.direction)` — a method that, per this file's own §7 pitfalls list
("`this.gameId`, `engine.pullToward`, `engine.movePlayer` do not exist"),
**never existed**, the same invented-API failure mode documented there for a
different method. It crashed roughly 1 in 50 games in `test_headless.js`
(4/200) — reachable only through the AI character-action path, which explains
why it survived undetected: `test_ai_vs_ai.js`/`test_ai_difficulty.js` don't
exercise character actions at all (§6), so only the one suite that actually
drives the real `GameController` ever hit it. Fixed to match the correct,
already-working human-side pattern (`ui.js`'s `runCharacterAction()`):
`CHARACTERS[player.characterKey].action.resolve(this.engine, player, opp,
action.direction)`.

**Full verification after all six fixes (five pending + the newly found bug),
run for real via `node`, not by hand**: `node -c bundle.js` clean;
`test_ai_vs_ai.js` 0/100 crashes; `test_headless.js` 0/200 crashes (was 4/200
before the `doCharacterAction` fix); `test_ai_difficulty.js` 0 crashes, both
ladder checks `true`; `test_cammy_zangief_fix.js` 16/16;
`test_cammy_zangief_games.js` 0 crashes across all three matchups (this file
was later consolidated into `test_ai_vs_ai.js --focus`, see `CHANGELOG.md` —
this historical note is accurate for the point in time it describes). **Note:
`verify_howtoplay.js`, referenced elsewhere in this file, does not actually
exist in this project snapshot** — running it errors with `MODULE_NOT_FOUND`.
That's a stale doc reference, not a regression; harmless to leave uncorrected
elsewhere in this file, but don't waste time looking for a file that isn't here.

**Environment note for a future session that also can't find Node**: it may be
installed but not on PATH. Check `C:\Program Files\nodejs\node.exe` (or run
`Get-ChildItem -Path $env:ProgramFiles -Filter node.exe -Recurse` in
PowerShell) before concluding it's unavailable. The documented build process
below uses `python3`; if Python isn't available either, the same
concatenate-and-substitute logic is trivial to reimplement directly in Node
(read the three files, strip each one's `module.exports` block with a
**global**-flagged regex — a first pass here used a non-global regex and
silently left a second `require()` destructure line in `ai.js` un-stripped,
which then threw `Identifier 'CHARACTERS' has already been declared` — write
the concatenation to `bundle.js`, then string-replace `shell.html`'s three
placeholders and write `exceed_poc.html`).

Single-file HTML/JS implementation of **Exceed**, a 2-player tabletop fighting
card game, human-vs-CPU. Vanilla JS, no build step to *play* — open the HTML
file in a browser. Editing requires a bundling step (see §2).

Ground-truth rules sources (should be in the project's knowledge/files):
`Exceed_How_to_play`, `Exceed__The_Comprehensive_Rulebook.txt`,
`Commonly_Misunderstood_Rules` (read closely — several engine decisions exist
because of specific clauses in it), `EXdata.xlsx` (authoritative stats/card
text for every fighter, implemented or not — a `Template` sheet documents the
column layout). **Player-facing card text in cards.js should be transcribed
from this workbook, not paraphrased** — implementation notes belong in code
comments, never in the string shown to the player.

## 1. Practical notes for a different AI picking this up

- Development happens in a scratch directory not visible to the user; the
  `.js` files are the real source — **edit those, never hand-edit
  bundle.js/exceed_poc.html directly.**
- If your environment mounts project files read-only (e.g. a Claude Project
  snapshot), your edits do NOT save back — the only durable output is
  whatever you write to the actual outputs location and hand back to the
  user. Don't assume a later session will see in-place edits.
- Most of this project is verified headless only (Node + a DOM stub), so
  passing tests validate *logic*, not rendering fidelity. Earlier Playwright
  sessions covered a few pieces (fighter-select cards, fighter-info modal,
  deck-reference modal, Axl's hand Cancelable badges), and **2026-08-16 added
  a full real-Chrome playtest** of the built `exceed_poc.html`: every
  interactive `G.mode` added in the Tier 1-4 work was reached in a real
  browser and visually checked, at four viewport sizes, with zero page or
  console errors (see `CHANGELOG.md` for that session's findings). The
  harness that did it is not checked in — it lived in a scratch directory —
  but it is easy to rebuild: `playwright-core` driving installed Chrome
  (`channel: 'chrome'`, no browser download), a copy of `test_headless.js`'s
  decision switch running *in the page* instead of under `vm`, pausing so the
  Node side can screenshot a given mode. Two things it must do that are easy
  to get wrong, both learned the hard way:
  - `pendingChar`/`pendingAiChar` in `shell.html` are script-scope `let`s, not
    `window` properties, so **you cannot start a specific matchup by assigning
    to `window.pendingChar`** — it silently keeps playing whatever the select
    screens last set (a first pass at this quietly played the same matchup ~20
    times while reporting different ones). Click through the real select
    screens, which also means one page reload per game, since `render()`
    replaces all of `#app` — the select screens included.
  - **(Fixed 2026-08-17 — kept here because the reasoning still applies to any
    NEW persistent UI.)** `openHowToPlay()` used to work only before a game
    started, because the button and `#howToPlayOverlay` both lived inside
    `#app`, which `render()` replaces wholesale. Both now live OUTSIDE `#app`
    as siblings, so the rules are reachable mid-game. Anything else that must
    survive a repaint has to go outside `#app` the same way — putting it in
    the paint tree means it exists only until the next action.
- Opening the file in a real browser before declaring UI work done is still
  the highest-value habit here; character-specific modes gated on rare states
  (Millia's/Leo's after-strike hooks need the opponent *stunned*) will not
  show up in random play — bias the driver toward the action that reaches
  them, or they simply never appear.
- Read §7 (invariants) and §8 (pitfalls) before writing any engine code —
  they document non-obvious constraints that aren't discoverable by reading
  the code alone, and violating them breaks things silently.

## 2. Source of truth: files on disk

```
cards.js       — card data + effect hooks (NORMALS + GG_NORMALS pools, all characters' cards, CHARACTERS)
engine.js      — rules engine: Player/state, movement, strike resolution (generator-based)
ai.js          — AI: Monte Carlo strike selection, non-strike action heuristic
ui.js          — GameController: turn flow, human interaction handlers, drives the strike generator, AI turn execution
render.js      — pure DOM rendering (no game logic) — reads GameController state, builds HTML
shell.html     — HTML page shell + CSS + character-select screen + placeholders __BUNDLE__/__UI__/__RENDER__

bundle.js       — GENERATED: cards.js+engine.js+ai.js concatenated, Node-isms stripped. Regenerate, don't hand-edit.
exceed_poc.html — GENERATED: shell.html with placeholders filled in. THIS is the deliverable. Regenerate after any source change.

test_ai_vs_ai.js      — AI-vs-AI full games via vm; hit rates / mechanic usage counts.
                        `--focus` (or `--focus=charKey:oppKey`) runs an opt-in
                        focused per-character A/B comparison instead — see §5.
test_headless.js      — full human(scripted)-vs-AI games via the REAL GameController+render() via Node's vm
test_ai_difficulty.js — validates Champion/Warrior/Student form a real strength ladder
test_cammy_zangief_fix.js — unit-level assertions pinning exact numeric behavior for
                        specific cards (Flying Power Bomb, Razor's Edge Slicer, etc.) —
                        the only place that would catch a regression in the specific
                        range-legality bug those assertions target; the other suites
                        only check aggregate hit-rate/crash counts.
test_blazblue.js      — the same idea for Season 5: pins the two shared mechanics
                        (Overdrive fill/tick/Revert, Astral Heat) and every implemented
                        BlazBlue fighter's Innate/Exceed/Overdrive. Add a section here
                        when promoting a new Season 5 fighter — the crash-count suites
                        cannot tell a working passive from a silently dead one.
test_ai_planner.js    — unit assertions for ai.js's three-turn lookahead (planThreeTurns
                        and the state/evaluation pieces under it), plus a per-decision
                        time budget. Every assertion in it is an invariant that was
                        WRONG in an earlier draft and cost measurable win rate — read it
                        before changing PLAN_CONFIG.
test_guiltygear.js    — the same idea for Season 4: pins every GGST fighter's
                        Innate/Exceed/character action plus the card-level primitives
                        their data added (hand-size clauses, boost-count Range scaling,
                        seal-after-use, face-up-per-card). Add a section here when
                        promoting any further GG fighter.
verify_howtoplay.js   — extracts built HTML's <script> blocks, DOM-stubs them, drives openHowToPlay()/showHowToPlayTab()
```
There is no `test.js`/`test_ai.js` — stale names from an old draft, ignore if seen elsewhere.
There is no longer a `test_cammy_zangief_games.js` — its per-focus-character
stat tracking was consolidated into `test_ai_vs_ai.js`'s `--focus` mode
2026-08-14 (it used to duplicate that file's entire turn loop just to add
this). See `CHANGELOG.md` for detail; if you see it referenced anywhere else
in this doc, that's stale.

### Build process (re-run after ANY change to cards.js/engine.js/ai.js/ui.js/render.js)

```bash
cd /home/claude/exceed
python3 << 'EOF'
import re
def clean(src):
    src = re.sub(r"const \{[^}]*\}\s*=\s*\(typeof module[\s\S]*?;\n", "", src)
    src = re.sub(r"\n?if \(typeof module !== 'undefined'\)[\s\S]*$", "\n", src)
    return src
with open('cards.js') as f: cards = f.read()
with open('engine.js') as f: engine = f.read()
with open('ai.js') as f: ai = f.read()
bundle = "// ===== cards.js =====\n" + clean(cards) + "\n// ===== engine.js =====\n" + clean(engine) + "\n// ===== ai.js =====\n" + clean(ai)
with open('bundle.js', 'w') as f: f.write(bundle)
EOF
node -c bundle.js && echo "bundle OK"

python3 << 'EOF'
with open('bundle.js') as f: bundle = f.read()
with open('ui.js') as f: ui = f.read()
with open('render.js') as f: rnd = f.read()
with open('shell.html') as f: shell = f.read()
out = shell.replace('__BUNDLE__', bundle).replace('__UI__', ui).replace('__RENDER__', rnd)
with open('exceed_poc.html', 'w') as f: f.write(out)
EOF
cp exceed_poc.html /mnt/user-data/outputs/exceed_poc.html
```

Then run the test suite (§5) before considering any change done. **The
bundler's exports-stripping is regex-based and assumes the `if (typeof
module !== 'undefined') module.exports = {...}` block is the LAST statement
in each source file** — code added after it gets silently deleted. Keep that
block last, or update the regex.

## 3. Architecture

- **cards.js**: each card is a plain object with base stats
  (`range`/`pow`/`spd`/`arm`/`grd`), a `hooks` object (`before`/`hit`/`after`,
  plus flags like `interactiveHit`, `rangeMaxBonusIfInitiating`,
  `spdBonusIfInitiating`), and a `boost` object (`force` cost, `continuous`,
  `resolve()`, plus flags like `needsNameChoice`, `triggersStrike`).
- **engine.js**: `Engine` holds both players' full state (see `makePlayer()`).
  Strike resolution is `*resolveStrikeSteps(...)`, a **generator** that only
  `yield`s at points needing a real mid-combat decision (Grasp's push,
  Block's Force-for-Armor, Hadoken's recycle choice, sustain choice, and
  several Axl-era yields — see §7 invariant #2 for the full current list).
  Every other card resolves synchronously in one tick — deliberate design
  choice per explicit user instruction ("keep resolution quick where no added
  complexity is needed"). Do not convert every hook into a generator "for
  consistency."
- **ai.js**: (1) `chooseAction(...)` — non-strike action heuristic. (2)
  `chooseStrikeSelection(...)` — Monte Carlo: samples plausible opponent hands
  from the unseen pool, scores via `evalMatchup(...)`, and picks by solving a
  mixed strategy over the payoff matrix. (An older `chooseStrikeCard(...)`,
  softmax over a single best-response assumption, was superseded by it and
  deleted 2026-08-17 — it had no callers left.) (3) `resolveAIStrikeRequest(...)` answers
  the engine's generator yields for the AI side. **No-cheating invariant**: no
  function here ever receives the opponent's real `hand` array, only
  `publicInfoOf()` snapshots.
- **ui.js**: `GameController` class, owns `this.engine`/`this.mode`. Human
  actions are methods (`actionPrepare()`, `pickStrikeCard()`, etc).
  `driveStrikeGen(gen, sendVal)` loops `gen.next()` — pauses for a human
  yield (stores `this.pendingGen`), resolves AI yields immediately via
  `resolveAIStrikeRequest()`.
- **render.js**: pure functions reading the global `GameController` (`G`),
  producing an HTML string for `#app`. No state mutation here — ever.

## 4. Current character roster status

**Fully playable, no known blockers**: ryu, ken, cammy, zangief, axl, sol
(Sol Badguy), cviper (C. Viper), vega, akuma, chunli (**Innate fixed
2026-08-16** — see below), dan (**Innate fixed 2026-08-16**, one small Exceed
gap remains, see below), mbison (**fixed 2026-08-16** — see below), leo (Leo
Whitefang — Innate/Exceed + turn-end Revert fully implemented), ky
(**promoted 2026-08-16** — Cancel-based Innate/Exceed fully wired end to end;
his card data was replaced with the real workbook data 2026-08-17, see the
Season 4 paragraph below),
testament (**promoted 2026-08-16** — see §7's `normalsHitHook` fix), potemkin
(**promoted 2026-08-16** — see §7's `rangeConditionalStatBonus` fix), millia
(**promoted 2026-08-16** — see §7's `named_choice_effect` fix), sagat
(**promoted 2026-08-16** — see §7's face-up Strike mechanic fix).

**Season 5 (BlazBlue)**: the season's two shared mechanics (Overdrive, Astral
Heat — see the header comment above `RAGNA` in cards.js) and the season's
engine-side primitives were built in one pass, and card data was transcribed
from `EXdata.xlsx` for nine fighters at the same time.
**Fully playable**: ragna, jin, noel, taokaka; plus **added 2026-08-17** —
tager (Iron Tager), nu13 (ν-No. 13), bang (Bang Shishigami), hakumen,
platinum (Platinum the Trinity).
`platinum` (Platinum the Trinity) was **finished 2026-08-17** and is fully
playable. Her Innate and Exceed are both "Cleanup: you may reveal the top card
of your deck; if it has a Continuous Boost, play that boost [then sustain it]",
which needed the character-level Strike-cleanup hook that did not exist:
`CHARACTERS[key].onStrikeCleanup(engine, player, opponent, info)`, a generator,
dispatched from `resolveStrikeSteps` AFTER `cleanupBoosts` and
`triggerCardCleanup` and BEFORE `onAfterStrike`. That ordering is load-bearing:
a boost this hook plays must not be swept by the sweep that just ran, and the
Exceed's "sustain one of your Continuous Boosts" must see what survived.
Playing the revealed boost does NOT go through `playBoost` (no Force cost, no
`triggersStrike` follow-up is possible from cleanup) — it uses
`engine.putContinuousBoostIntoPlay`, factored out of
`reviveBestContinuousBoostFromDiscard`, whose `runImmediate` option is what
distinguishes "PLAY that boost" (Platinum, fires the boost's "Now:" clauses)
from "put it into play" (Sky High Claw, which deliberately does not).
"Then sustain it" is a one-shot `_sustainOnce` flag on the in-play boost entry,
consumed by `cleanupBoosts`. Both of her prompts reuse the generic
`named_choice_effect` yield rather than adding UI modes.
Every other BlazBlue fighter in the workbook (Arakune, Carl Clover,
Hazama, Kokonoe, Litchi Faye-Ling, Nine the Phantom, Rachel Alucard) has no
card data in this project at all yet.

**Season 4 (Guilty Gear Strive)**: finished 2026-08-17 — all thirteen remaining
GGST fighters are now selectable with real card data: chipp, ky, anji, may,
ramlethal, nagoriyuki, baiken, faust, giovanna, ino, jacko, happychaos,
goldlewis (plus axl, sol, testament, potemkin, leo, millia, which were already
done). **The card data for chipp/ky/anji/may/ramlethal that existed before that
session was invented, not transcribed** — wrong card names, wrong stats, wrong
text, wrong Exceed costs on three of them — written in a session where
`EXdata.xlsx` was not present. It has been replaced wholesale from the workbook.
If you ever find yourself about to write card data from memory, don't: leave the
character unfinished instead, exactly as the Nagoriyuki note here used to say.

Per-character clauses deliberately left unimplemented are marked `// DEFERRED:`
on the card itself in cards.js and are all small riders, never a character's
core identity. The recurring ones, worth knowing before adding more cards:
- **Scaling Gauge spends** ("Spend up to [N]. For each spent, +1POW") — the
  optional-spend yields are Force-based or fixed-amount Gauge; nothing scales a
  Gauge payment. Chipp's Zansei Rouga, Hakumen's Guren, Iron Tager's Ultras.
- **Mid-Strike Armor changes** ("After: Lose all Armor") — `liveStats` computes
  Armor once, up front, on purpose (see `selfArmorZero`'s comment for the
  ordering bug that proved a hook is the wrong place). I-No's Ultimate
  Fortissimo, Jack-O's Chain of Chiron, Goldlewis's Slam.
- **Boosts that affect both players' Before/After steps** (Faust's item boosts)
  — the only cross-player boost field is `opponentPowMod`.
- **"You may spend {N}. If you do, Strike."** — `triggersStrike` is
  unconditional, so the follow-up Strike is offered even when the optional
  Force went unpaid.

`zato1` was **finished 2026-08-17** — the Eddie token system is built, and Seth
is now the only entry left in `shell.html`'s `UNFINISHED_CHARACTER_NOTES`.

**The Eddie system, since it is this project's only third board entity.** Eddie
is `player.eddiePos` (`null` = out of play, otherwise a space); every other
character leaves it null forever. He is explicitly "not a character and is
ignored when moving", so **no movement code reads that field** — he never
blocks, is never passed through, is never pushed. Only two things read him: the
range calculation and his owner's own card text. The engine section
(`usesEddie`/`eddieInPlay`/`placeEddie`/`removeEddie`/`eddiePlacementSpaces`/
`attackSourcePos`/`attackRange`/`pushAwayFromAttackSource`) is entirely
data-driven off `CHARACTERS[key].usesEddie` + `.specialsRangeFromEddie`, so
nothing outside cards.js knows Zato-1 exists. The one change to shared code is
in `resolveStrikeSteps`: the Strike's range check calls
`engine.attackRange(attacker, defender, attCard)` instead of
`engine.range(attacker, defender)` — identical for everyone else, Eddie's
distance for a Zato-1 Special/Ultra. His three character hooks are
`normalsAfterHook` (new; the Normals-gated generator sibling of
`attackAfterHook`, which is what places Eddie), `attackHitHook` (the Advantage)
and `onStrikeCleanup` (the removal, skipped for a card with
`keepsEddieOnCleanup`). Interactivity: the `place_eddie` Strike yield →
`place_eddie_choice` UI mode, and the Boost-time `needsEddieSpaceChoice` →
`eddie_space_choice` mode (a generic board-space picker — Vega's Scarlet Terror
boost, still parked on the unimplemented `needsAnySpaceChoice`, could reuse it;
what it still lacks is the "move to any space and Strike + innate-bonus
tracking" half, not the picker). AI: `chooseEddieSpace` in ai.js, and
`evalMatchupDetail`'s optional `eddiePositions` param.
Two smaller known gaps that
did NOT justify the unfinished badge, in the same spirit as Dan's and Vega's
already-accepted stubs: Giovanna's Exceed action still isn't charged its [1]
(`actionForceCost` is flat/always-charged and her Innate action is genuinely
free), and Happy Chaos's Exceed hands him Deus Ex Machina and offers the Strike
rather than forcing that specific card to be set (the strike-selection path has
no "you must set this card" shape).

**AI mechanic-coverage audit, 2026-08-16** (the follow-up this file's own §6
called for, beyond Ryu/Ken/Zangief/Cammy/Vega/Akuma/C.Viper): found three
characters that were listed "fully playable" but had a COMPLETELY
unimplemented Innate/Exceed (zero matches for the character outside
cards.js, confirmed via grep — not a documentation nuance, an actually
non-functional passive a player could have picked):
- **M. Bison** ("Action: Add [up to 3] card(s) from hand to Gauge, draw that
  many; Exceed may take this immediately") — **FIXED.** New
  `needsHandToGaugeChoice`/`maxHandToGauge` action shape (the generic
  `action.resolve(engine, player, opponent, direction)` signature can't
  express a multi-card-selection choice), dispatched via a dedicated
  `action_gauge_select` UI mode instead of the usual `action.resolve` path.
  `onExceed` returns `{mayAction: true}`, consumed by a new
  `char_exceed_action_choice` mode (mirrors the existing `mayStrike`/
  `char_exceed_strike_choice` shape). AI: new `aiRunHandToGaugeAction`
  (picks lowest-POW cards, same convention ChangeCards' AI heuristic uses) +
  a `scoreCharacterAction` branch. **Also fixed a related pre-existing gap
  while here**: the AI's `'exceed'` action-type branch in `runAITurn` used to
  call `engine.doExceed(player)` and completely discard its return value —
  meaning Anji/May's own `onExceed` `mayStrike` bonus was never followed up
  by the AI either, even though the human-side `actionExceed()` already
  handled it. Both `mayStrike` and the new `mayAction` are now honored on
  the AI side too.
- **Chun-Li** ("Innate — After you resolve a Boost, you may Retreat 1")
  — **FIXED.** Only her Exceed half (RNG-per-continuous-boost, via
  `normalsRangeBonus`) was live; the Innate half existed only as a
  commented-out `// mayRetreatAfterBoost: true` field with no yield behind it
  despite a comment claiming otherwise. New `mayRetreatAfterBoost: true` +
  `retreat_after_boost_choice` yield (human) / unconditional-take heuristic
  (AI, "no real downside identified" — same shallow-heuristic level as
  `sustain_choice`), inserted into `doBoost`'s continuation chain BEFORE any
  follow-up Strike, per her own printed clarification ("If you would Strike,
  retreat first").
- **Dan** ("Innate — Your Wild Swings have 'Critical: +1POW and +1SPD'")
  — **FIXED.** New `wildSwingCriticalStatMods(player)` extension point,
  gated on the pre-existing `fromWildSwing` parameter `liveStats` already
  threads through (Leo's `wildSwingStatMods` established the precedent).
  Also threaded `fromWildSwing` through `ai.js`'s
  `hasRealCriticalValue`/`criticalPayProbability`/`chooseCriticalPayment` and
  `ui.js`'s `aiDecideCritical` (previously dist-only) so the AI correctly
  values paying Critical on its own Wild Swing instead of defaulting to the
  low bluff rate. **Not fixed, real remaining gap**: his Exceed's "may draw
  from top or bottom of deck, then may discard" — a comment claiming this was
  "handled in ui.js passOrContinue with mode 'deck_choice_draw'" was
  aspirational; that mode doesn't exist (confirmed via grep). Left honestly
  documented rather than silently claimed done; Dan stays in the
  fully-playable bucket since this is a minor missing flourish, not his core
  identity (same precedent as Vega's already-accepted Scarlet Terror stub).
- **Guile** ("Innate/Exceed — Change Cards Gauge-into-Force conversion") —
  **FIXED 2026-08-17; he is fully playable and out of
  `UNFINISHED_CHARACTER_NOTES`.** Two sessions recorded this as "genuinely
  ambiguous without the source rulebook". The rulebook was on disk the whole
  time (`D:\stažené\Exceed_ The Comprehensive Rulebook.txt`, plus
  `Exceed_Rules_Final.pdf`) and answers it outright: Change Cards is "spend
  Force to draw an equivalent number of cards", and Force is generated by
  discarding — Normal/Special 1, Ultra 2, **Gauge card 1**. So the payment this
  Innate modifies does exist, a Gauge card is a legal source, and "{2}" is 2
  more Force, which in this action means 2 more cards drawn. Change Cards was
  reworked to match (`engine.performChangeCards`); Guile's hooks are
  `changeCardsForceBonus` (flat +2, once per action — the text says the FIRST
  Gauge spent) and `mayStrikeAfterChangeCards` (his Exceed's follow-up Strike,
  offered via the existing `may_strike_choice` mode). **Lesson worth keeping:
  before recording a mechanic as undecidable, check whether the source document
  is actually available — twice here it was, and wasn't looked for.**

**Historical note, superseded 2026-08-17**: an earlier version of this section
listed nagoriyuki/chipp/anji/may/ramlethal/baiken/ino/faust/jacko/giovanna/
happychaos/goldlewis as "card data implemented but Innate/Exceed blocked or
simplified". All twelve are finished now (see the Season 4 paragraph above) —
the blocker for most of them was that `EXdata.xlsx` wasn't present in those
sessions. It is now in the project directory; use it.

**Explicitly excluded from CHARACTERS entirely**: `seth` — per direct user
instruction. Do not re-add without asking.

Full roster list lives in `CHARACTERS` in cards.js; treat the above as a
snapshot, not a substitute for checking the actual object.

## 4b. Major implemented systems (functionally complete, tested) — reference

- **Card typing is load-bearing — never type a character's card `'normal'`
  (2026-08-17).** `'normal'` describes the shared blue pool only (`owner:
  'shared'`, plus `GG_NORMALS`); a character's own cards are `'special'` and
  `'ultra'`. Three engine behaviours read `type` directly: `payUltra` charges
  Gauge only for `type === 'ultra'` (so a Gauge-cost card typed `'special'` is
  **free to Strike with** — this was live for 16 characters), `forceValueOfCard`
  makes Ultras worth 2 Force, and every "your Normals ..." passive
  (`normalsStatMods`, `normalsRangeBonus`, `normalsHitHook`,
  `grantsNormalsBeforeAdvance`, `sustainIfNoNormalUsed`) gates on
  `type === 'normal'`. A workbook cost in column B (`[N]`) means the card is an
  Ultra. `test_changecards_guile.js` now asserts this roster-wide.

- **Beginner onboarding (2026-08-17)** — built on one rule: teach from the
  mechanics, never from prose. Four pieces, each anchored to something the
  engine already does, so none of them can drift out of sync with the rules:
  1. `engine.lastStrikeInfo` — every Strike recorded as data (printed vs. live
     stats, speed order, then ordered `activate`/`damage`/`stun`/`cleanup`/
     `next` events). Write-only from the engine's side; a rendering feed like
     `fxQueue`, never read by game logic. **If you add a new disposal branch to
     `cleanupOne`, or a new miss condition to the range check, record it here
     too** — a card that vanishes with no recorded destination shows up in the
     UI as an unexplained gap.
  2. `MODE_HELP` in render.js — per-decision help keyed on `G.mode`. Adding a
     new interactive mode without a `MODE_HELP` entry is not a crash, just a
     silently unexplained prompt; add the entry with the mode.
  3. Coach-tip persistence in ui.js (`coachSeen`/`markCoachSeen`/
     `coachExpanded`/`toggleCoach`). Lives in ui.js specifically because
     render.js must stay pure — "seen" is marked when the player LEAVES a mode,
     not when the tip is first painted, since modes repaint repeatedly
     mid-decision.
  4. Beginner mode (`G.beginner`, from `GameController`'s 4th `opts` arg) —
     gates the Critical prompt until damage has gone both ways, paces the
     strike breakdown one beat at a time, keeps coach tips expanded. See §8
     invariant #12: beginner mode may only change presentation and skip
     prompts whose skip is a no-op; it may never change a rule.
  Plus `G.suggestMove()`, which runs ai.js's `chooseAction` on the human's own
  position at `randomness: 0` — the teaching tool and the opponent share one
  evaluator on purpose, so a hint can never contradict the game.
- **Full turn loop**: Prepare, Move (paid, `advanceSelfPaid`/`simulateAdvance`,
  player picks which cards pay Force via `move_force_select`, hand+gauge
  both eligible via `engine.payForceFromHandOrGauge`), Change Cards (player
  picks which cards via `changecards_select`, not auto-picked), Boost,
  Exceed, Reshuffle, character actions.
- **Mulligan** (`engine.performMulligan(player, setAsideUids)`): before turn
  1, set aside any number of opening cards face-down, draw replacements
  first, then shuffle set-aside back in. Separate mechanic from
  `manualReshuffle()` (mid-game, once per game, different rule despite
  similar name). AI heuristic: `ai.js`'s `chooseMulliganSetAside`.
- **Criticals**: setting any attack card, either side may pay 1 Gauge
  (`engine.canPayCritical`) to activate that card's `critical` field
  (statMods/before/hit/after/ignoreArmor/ignoreGuard/rangeImmuneRanges/
  stunImmune) for that Strike only. Wild Swing critical prompts are blind
  (card not shown). `liveStats` folds Critical stat mods in before Speed
  order is determined. AI: `chooseCriticalPayment` (~85% pay on a real
  Critical, ~8% bluff otherwise) — a simple heuristic, not full game theory;
  `evalMatchup` does not model Criticals at all (see §6 gap).
- **EX Strikes** (pair two same-key cards for +1/+1/+1/+1, second discarded),
  **Grasp** (interactive push, generalized `grasp_move` yield with
  `min`/`max`/`forceDirection` from the card — reused by Cannon Spike/CQC/
  Gyro Drive Smasher), **Block** (reactive Force-for-Armor), **Focus**
  (Reading — names a Normal, forces a defend-or-reveal, also grants
  `_focusActive` movement immunity for that activation only).
- **Two Normals pools**: `NORMALS` (Street Fighter cast) and `GG_NORMALS`
  (Guilty Gear cast, Axl onward) — `CHARACTERS[key].normals` selects, falling
  back to `NORMALS`. Anything enumerating "the Normals" must use
  `char.normals || NORMALS` (§8 invariant #10).
- **Multi-fighter data-driven extension points** (never branch on
  `characterKey === '...'` outside cards.js): `CHARACTERS[key].action`
  (per-turn character-action button, shape `{needsDirection, resolve(engine,
  player, opponent, direction) -> {mayStrike}?}`, omit entirely for
  passive-only characters), `.onExceed(engine, player, opponent)`,
  `.silhouettePose`, `.blurb`, `.sideSwapRangeInclude`/
  `.exceededSideSwapHit(ctx)` (Cammy), `.extraCriticalStatsIfOpponentAtRange1`
  (Zangief), `.normalsStatMods`/`.normalsRangeBonus` (Axl). Per-card:
  `rangeImmuneRanges` (base/critical/boost), `movementImmune`,
  `blocksOpponentPassThrough`, `critical.stunImmune`/`boost.stunImmuneGrant`,
  `boost.sustainIfNoNormalUsed` (real yes/no yield, `sustain_choice`),
  `boost.expireToGaugeAfterTurns` / `expireToGaugeAtStartOfNextTurn` (two
  different timings — end vs. start of turn), `boost.toGaugeIfHitThisStrike`
  (mandatory, no prompt), `boost.grantsNormalsBeforeAdvance`,
  `boost.rangeMaxBonus`, `boost.nowClose`/`nowDraw`.
- **Extension points added by the Season 4 (GGST) finish pass, 2026-08-17** —
  all deliberately generic, so the next character needing the same shape adds no
  engine code. Per-card: `liveStatMods(owner, opponent, engine, info)` (ONE
  conditional stat-mod hook replacing what would have been five narrow fields —
  Baiken's hand-size clauses, Ky's "Range 2: +2SPD", Faust's "if you have a Boost
  in play", Goldlewis's "if you initiated"; folded into `liveStats` before Speed
  order like every other stat mod), `dynamicRangeMaxBonus(owner, opponent,
  engine, isInitiator)` (Jack-O's per-Boost Range scaling),
  `hitImmuneToFasterAttacks` (reuses the existing `_unhittableThisStrike` flag,
  decided right after Speed is final), `sealAfterUse` and `neverLeavesPlay`
  (both handled in `cleanupOne`'s disposal branch), `canInitiateFaceUp`.
  Per-character: `invalidStrikeCard(player, card)` (replaced the one
  `characterKey === 'nagoriyuki'` branch that was left in `engine.js`),
  `cardCleanupFor(player, card)` (returns the same shape a card's own
  `cleanup` field uses), `onReshuffle(engine, player)`, `canExceedAction:
  false` + the new `engine.forceExceed(player)` (Exceeding without the action),
  and `canGoFaceUp` may now be a predicate on the card rather than a boolean.
  `player.revertPending` finally does something: it is consumed at the end of
  `resolveStrikeSteps`, which is what "Revert during cleanup" means.
- **Two real bugs the same pass found in code that predates it**:
  `buildDeck` read `c.copies || 2`, so a deliberate `copies: 0` (Happy Chaos's
  Deus Ex Machina, which is never a deck card) silently became 2; and
  `triggerOnCancelHooks` picked `char.onCancelExceed` whenever the player was
  Exceeded, with no fallback — Chipp and Ky both branch on `player.exceeded`
  INSIDE `onCancel` and declare no `onCancelExceed`, so **both of their Exceeds
  had been dead the whole time**. Neither was reachable by the crash-counting
  suites; `test_guiltygear.js` caught both.
- **Three-turn lookahead planner** (`ai.js`, `PLAN_CONFIG`/`planThreeTurns`, added
  2026-08-17): `chooseAction`'s decision point now runs a my-turn → their-turn →
  my-turn search and returns the first action of the best line; the pre-existing
  one-turn heuristic below it is untouched and still runs as the fallback (no usable
  opponent sample, empty hand, or `usePlanner: false`). Structure, and why it is not
  the noise amplifier §6 used to warn about: **all** Monte Carlo sampling happens once,
  up front, in `buildPlanContext`, which builds two small memoized tables — my best
  exchange at a distance (max over my cards of min over their answers) and the mirror
  of it from their side — and the tree search is then pure arithmetic over those
  tables. Depth costs table lookups, not nested sampling. Three invariants matter and
  all three were wrong at some point during the build:
  1. **An opponent turn is a cost, never a bonus.** `evalMatchup` gives both sides a
     flat tempo bonus for connecting, so both players can score positively in the same
     exchange — using my own defensive value as the opponent-turn term therefore made
     "stand where they can hit me" score as a POSITIVE. Their turn is now priced as
     minus what it is worth to THEM (`oppInitiateValue`), clamped so it can never be
     positive. Fixing this alone moved the head-to-head from 47% to 55%.
  2. **Damage is priced exactly once, through life**, by `planEvaluate`'s `life`
     weight. An earlier draft also added the raw exchange value as an immediate gain,
     which put damage on a completely different scale from every positional term and
     collapsed the planner into a greedy striker (57% of all turns were Strikes).
  3. **Spending a card is charged for** (`usedAttacks`). The exchange table always
     reports the root hand's best attack, so without a decay a plan that Strikes twice
     gets credited with its best card twice and Striking looks free.
  Measured with `node test_ai_vs_ai.js --plan-ab=600`: **56% / 44%** against the
  legacy heuristic, average life remaining 6.4 vs 4.3. Re-run that after any weight
  change — the weights are tuned, not derived, and several plausible-looking values
  measured WORSE (see `CHANGELOG.md` for the ones already tried and rejected).
- **AI difficulty ladder**: `AI_DIFFICULTIES = {champion:0, warrior:0.20,
  student:0.40}` randomness — that fraction of individual AI decisions
  (action choice, strike/defense selection, Grasp/Block/Hadoken answers,
  naming, Critical payment) is replaced with a uniform-random legal choice.
  `new GameController(humanChar, aiChar, difficultyKey)`.
- **AI quality passes on top of the base heuristic**: life-aware
  `chooseAction` (presses when behind, eases off ahead), non-coin-flip
  Exceed timing, `scoreBoostCard`/`scoreNamedCardKey` (replaced uniform-
  random choices), Wild Swing priced as a real option by
  `wildSwingExpectedValue()` inside `chooseStrikeSelection` (this used to read
  `shouldWildSwing()`, a threshold plus a 15% dice roll that could override the
  priced comparison; removed 2026-08-17),
  `estimateBoostTimingValue`/`quickBestStrikeValue` (cheap same-turn
  strike-now-vs-boost-now comparison — cost-disciplined on purpose, see §6
  2-ply note for why), opponent hand-narrowing via
  `engine.recordHandReveal()` + `sampleOpponentHand()`'s exact-match
  short-circuit (kicks in after Focus/Block/Name-a-Range force a reveal,
  self-invalidates the moment any of 5 public counts changes), AI never
  selects an unaffordable Ultra (`buildStrikeCandidates` filters by
  `myGaugeCount` before both the smart and randomness-override paths).
- **Axl Low / Guilty Gear architecture** (foundational, not just his own
  card data): `buildDeck()` reads `CHARACTERS[key].normals` +
  per-card `copies` (default 2); `afterStunCheck(ctx, opponentWasStunned)`
  hook (fires once per exchange, only for the card that activated first,
  since only it can cause the second card's Stun); `cleanupBoosts` gained an
  `opponentWasStunned` param + `drawIfOpponentStunnedThisStrike` boost flag;
  `optional_gauge_spend`/`optional_retreat` yields; `peekTopN`/
  `resolveDeckLookChoice` (Sickle Storm's deck-peek, feeds into the same
  Strike machinery as a hand card via `{type:'decklook', card}`);
  `useBurstAction`/`player.extraActionsRemaining` (One Vision's "take 3
  actions," decremented in `passOrContinue`/`aiFinishNonStrike` before the
  normal turn-pass logic, unconditionally reset to 0 every Strike cleanup);
  `mustWildSwing(initiator, defender)` (GG Focus). `selfArmorZero` static
  field (NOT a before-hook — see §8 invariant #8, this is the bug that
  proved why).
- **Cancel mechanic** (`card.cancelable: true` on exactly the GGST-set cards
  the source sheet's 'C' tag marks): after resolving a cancelable Boost, may
  spend 1 Gauge to take another action, unlimited chaining falls out for
  free from the existing bonusAction loop-back (no counter needed). Modeled
  on the pre-existing `card.boost.bonusAction` mechanism (One Inch Punch),
  kept deliberately separate from `extraActionsRemaining` above — similar
  wording, different behavior, don't merge them. AI: `aiShouldCancel`
  (declines only if it would drop Gauge at/below Exceed reserve). Visual: a
  clock badge on cancelable cards. `test_ai_vs_ai.js`/`test_ai_difficulty.js`
  don't exercise it (see §6 known gap on those two files' simplified loops).
- **Fighter visuals**: `FIGHTER_PHOTOS[key]` (base64 data URI, single-file
  deliverable) takes priority over `SILHOUETTE_POSES` SVG pictograms in
  `fighterSilhouette()`. All supplied source photos so far have had an
  identical 8px white-strip defect along the bottom edge — assume any newly
  supplied photo has it too until checked, don't treat it as a one-off.
  Poster-style fighter-select cards (`fighterCardHTML()` in shell.html,
  shared by both the human-pick and CPU-pick screens so they can't drift
  apart) — a character with no photo gets its SVG pictogram dimmed/centered
  rather than looking broken.

## 5. Test suite — run before and after every change

```bash
cd /home/claude/exceed
node test_ai_vs_ai.js      # 100 AI-vs-AI games via vm; hit rate + mechanic usage counts
node test_headless.js      # 200 human(scripted)-vs-AI games via the REAL GameController+render()
node test_ai_difficulty.js # validates Champion/Warrior/Student form a real strength ladder
node test_ai_planner.js    # three-turn lookahead invariants + per-decision time budget
node test_ai_vs_ai.js --plan-ab=600   # planner vs the legacy one-turn heuristic, head to head
node test_blazblue.js      # Season 5 Innate/Exceed/Overdrive assertions
node test_guiltygear.js    # Season 4 (GGST) Innate/Exceed/action assertions
node test_onboarding.js    # beginner onboarding: lastStrikeInfo shape, coach tips, Critical gate, hints
node test_changecards_guile.js  # Force-based Change Cards, Guile's Innate/Exceed, roster-wide card-typing guard
node test_carddata.js      # every card in cards.js vs its EXdata.xlsx sheet; exits 1 on any mismatch
node verify_howtoplay.js   # run whenever shell.html's How to Play content changes
```

**`test_carddata.js` is the one to run after ANY card-data edit** (added
2026-08-17). It reads the workbook directly via `xlsx_reader.js` — a
dependency-free .xlsx reader, also usable on its own:
`node xlsx_reader.js EXdata.xlsx "Zato-1"` dumps a sheet, with no argument it
lists them — and compares Exceed cost, card names, Special/Ultra type, Gauge
cost, RNG/POW/SPD/ARM/GRD, the Cancel column and the printed Boost Force cost.
It found 37 real findings on its first run (four wrong Exceed costs, three
boosts implemented as free that print {1}, and a Cancel mechanic that was dead
for 15 fighters — see `CHANGELOG.md`). `node test_carddata.js text` additionally
prints wording differences, which are informational only: the Season 1-3 cards
use an older house style, the sheet itself contains typos the code corrects, and
printed "Critical:" clauses live in each card's `criticalText` field rather than
in `text` (a pass that "restored" them into `text` produced duplicates and had
to be reverted — check `criticalText` before believing one is missing).

`test_onboarding.js`'s reported check COUNT varies between runs (it plays a real
unseeded opening hand, and some assertions only apply once the scripted Strike
resolves) — only its pass/fail result is meaningful, never the total.

`test_ai_vs_ai.js` and `test_headless.js` must report **0 crashes**.
`test_headless.js` also asserts no `undefined`/`NaN` leaks into rendered HTML
and a minimum HTML length per render — treat failures as real bugs, not
flaky. `test_ai_difficulty.js` exits non-zero (WARNING) if the difficulty
trend doesn't hold — adjacent tiers (e.g. Champion vs Warrior) are genuinely
noisy in a single head-to-head due to real in-game luck; see §8 Pitfall
"ladder noise" before treating that specific comparison as a regression.

If you add a new `G.mode` value, add the matching `case` to
`test_headless.js`'s driver switch or it throws `Unhandled mode: ...` —
intentional, it's how the harness catches untested new states. This has
caught real bugs every time a new interactive UI mode was added (Axl's
session alone added five).

**Focused per-character A/B testing**: `node test_ai_vs_ai.js --focus` runs a
canned Zangief/Cammy/Ryu-mirror comparison (isolating one seated-as-p1
character's own hit/miss/out-of-range/Special-Ultra stats, since the default
aggregate run mixes both sides across random matchups). `--focus=charKey:oppKey`
runs any other pair, e.g. `--focus=vega:ryu`. Use this whenever a
character-specific AI fix needs a before/after comparison — don't write a new
duplicate turn-loop test file for it (a prior session did exactly that,
`test_cammy_zangief_games.js`, later consolidated back into this file's own
`playGame`/`doStrike`, see `CHANGELOG.md`).

## 6. Known open gaps / bugs — flat list

- **AI never uses manual Reshuffle.** `chooseAction()` (ai.js) has no branch
  for `engine.manualReshuffle(player)` — human-only in practice. **FIXED
  2026-08-16**: new branch in `chooseAction()` triggers Reshuffle when the
  AI's deck has ≤1 card left, Reshuffle hasn't been used, and the discard
  pile has ≥3 cards worth reclaiming (checked after the lethal-strike check).
- **`scoreCharacterAction` (ai.js) only scored `'ryu'` and `'ken'`. FIXED
  2026-08-16**: added scoring branches for baiken, ino, jacko, giovanna, and
  faust too (all mechanically functional today even though still UI-flagged
  unfinished for other reasons — see §4). Affordability-gated on
  `actionForceCost` where relevant.
- **Five duplicate copies of the unfinished-character key list in
  shell.html: FIXED 2026-08-16.** Consolidated into one
  `UNFINISHED_CHARACTER_NOTES` object + derived `UNFINISHED_CHARACTER_KEYS`
  array, referenced by all five call sites (`fighterCardHTML`,
  `renderCharSelect`, `renderAiCharSelect`, `selectCharacter`,
  `selectAiCharacter`).
- **Baiken/I-No/Jack-O's printed Force costs on their character action are
  not charged. FIXED 2026-08-16**: new `CHARACTERS[key].actionForceCost`
  field + a real paid-action UI flow (`action_force_select` mode in
  ui.js/render.js, hand-or-gauge picker, same shape as Boost/Move's; AI side
  pays via `pickForceUidsForTarget` and skips the action if it can't pay
  exactly). `test_headless.js` has a matching driver case.
- **Jack-O's Exceed "draw 3" is not wired. FIXED 2026-08-16**: `onExceed`
  hook added (`engine.draw(player, 3)`).
- **Giovanna's "Hit: Draw N" is not implemented. FIXED 2026-08-16** via a new
  `CHARACTERS[key].actionStrikeHitHook` primitive (narrower than Testament's
  `normalsHitHook`: gated on `actionStrikeThisTurn`, so it applies only to the
  ONE Strike her character action triggers, not every Normal). Still open:
  her Exceed action's [1] Force cost still isn't charged — unlike
  Baiken/I-No/Jack-O, her Innate action is genuinely free and only the
  Exceed version costs [1], an asymmetry `actionForceCost`'s flat
  always-charged shape can't represent without changing it for the other
  three too.
- **Dive's "cannot be hit" clause never implemented. FIXED 2026-08-16**: new
  `player._unhittableThisStrike` flag (mirrors the existing
  `_rangeImmuneThisStrike`/`_focusActive` per-strike-flag pattern) — Dive's
  `before` hook sets it on the attacker when the side-swap happens;
  `resolveStrikeSteps`'s hit-check reads it off the defender.
- **`evalMatchup()` (ai.js) doesn't model Criticals at all — STALE, already
  false as written.** `criticalGrant()`/`criticalPayProbability()` (added
  during the 2026-08-14 Akuma/Vega/C.Viper audit-fix work, generalized past
  those three characters) read any card's own `card.critical` field, not
  just character-specific extension points, and are blended into every
  Monte-Carlo-sampled card's expected pow/spd/arm/grd/range-immunity/
  stun-immunity — i.e. card PICKING already factors in Critical value for
  every character. Re-confirmed 2026-08-16. What's still genuinely
  unmodeled (narrower than this bullet implied): `critical.before/hit/after`
  *hook* side-effects (positional, not stat/damage — same class of gap as
  ordinary non-Critical hooks, see the hasMover() approximation note above
  `criticalGrant`), and Akuma's mutual grant's cross-player half (documented
  in `criticalGrant`'s own comment, deliberately not fixed — see there for
  why undervaluing it is the safe direction to be wrong in).
- **Zangief's Critical-gated immunities (Banishing Flat, Flying Power Bomb)
  are not modeled — STALE, already false.** Both use the standard
  `card.critical.rangeImmuneRanges`/`card.critical.stunImmune` shape, already
  covered by the generic mechanism above. Re-confirmed 2026-08-16.
- **Zangief's pass-through blocking (Siberian Blizzard,
  `blocksOpponentPassThrough`) is not modeled in `evalMatchup()`.** Confirmed
  2026-08-16: genuinely unmodeled, but investigated and found to have no
  measurable effect on scoring in the current architecture — `evalMatchup`
  only tracks 1-D *distance*, never which side of the board a mover ends up
  on, and its `getMoverSlack` floors effective distance at 0 either way
  (`Math.max(0, dist - slack)`), so "stopped adjacent" vs. "jumped past"
  produce an identical effective distance. Only matters if a future feature
  starts caring about board *side* again (à la Cammy's `sideSwapRangeInclude`
  or Vega's edge bonus) — not worth instrumenting before then.
- **`evalMatchup()`/`estimateBoostTimingValue()` don't model Cammy's sustain
  or boost-expiry timing.** Investigated 2026-08-16, left open: the timing
  estimator only ever looks one strike ahead (`quickBestStrikeValue` on the
  post-boost hand), so a boost's expiry timing rarely matters in practice —
  it never assumes multi-turn availability to begin with, so nothing is
  over-credited. The one real gap is Cammy's `sustainIfNoNormalUsed` (paying
  to indefinitely extend a boost) being invisible to this estimator, which
  would require deeper multi-turn simulation to value correctly; judged too
  speculative/high-risk-of-mistuning for the value, given this file's other
  heuristics are deliberately this shallow throughout.
- **~~2-ply lookahead: deliberately still not built~~ — BUILT 2026-08-17, and it
  went three turns deep, not two.** The prior analysis here argued a deeper Monte
  Carlo search stacked on a noisy per-ply evaluator would amplify noise rather than
  average it out. That objection was answered structurally rather than overruled: the
  planner samples ONCE per decision into a memoized table and searches over the table,
  so there is exactly one sampling stage no matter how deep the search goes. Two of the
  old bullet's suggestions were followed (reuse the existing cheap strike evaluator;
  approximate the resulting hand/position rather than tracking it exactly); the
  "only trigger it when the top two candidates are close" idea was NOT needed, because
  the table structure made a full search cheap enough to run every turn. See §4b for
  the design and the three invariants that were wrong during the build. Measured at
  56%/44% against the heuristic it sits on top of.
- **"No live opponent Gauge tracking" — narrower than it sounds, checked
  2026-08-16.** The opponent's Gauge is already full public information
  (`publicInfoOf`'s `gauge: player.gauge.map(c => c.key)`, exact card keys,
  not just a count) and IS already read for one purpose — the AI's own
  Ultra-affordability filtering uses `myGaugeCount` (invariant #13, own side
  only, deliberately never the opponent's). What's actually missing is
  narrower: no scoring function reasons about whether the OPPONENT can
  afford to pay a Critical/Ultra/EX in response, i.e. "this card is safer to
  set because they're Gauge-starved" isn't modeled. Real, but smaller in
  scope than "no tracking" implies — the data is already there, wiring it in
  is a scoring-heuristic addition, not a new plumbing/no-cheating-boundary
  problem.
- **Fixed 2026-08-15 (partial)**: Axl/C. Viper/Vega/Akuma/Sol Badguy now have
  real photos too (`axl.png`/`cviper.png`/`vega.png`/`akuma.png`/`sol.jpg`),
  same `FIGHTER_PHOTOS` mechanism as the original four — see `render.js`'s
  header comment there for full detail (PNG per explicit request for the
  first three; Akuma kept PNG and Sol kept JPEG, matching whatever format
  each source file already was rather than force-converting; all resized to
  480px wide, the largest this project ever actually renders a fighter photo
  at, before embedding; akuma.png's source had an unused alpha channel,
  confirmed fully opaque by sampling rather than assumed, dropped to 24bpp RGB
  during the resize; all five checked for and did NOT have the earlier four's
  white-strip defect). `FIGHTER_PHOTOS` now covers 9 of the 14 fully-playable
  fighters (§4) — guile/chunli/dan/mbison/leo still fall back to the abstract
  `SILHOUETTE_POSES` pictogram, no photo asset for them yet.
- **`test_ai_vs_ai.js`/`test_ai_difficulty.js` have their own simplified
  turn-loop reimplementations** that don't exercise several interactive
  mechanics even for finished characters: Cancel, deck-look, forced Wild
  Swing, push-pull choice. Only `test_headless.js` (drives the real
  `GameController`) exercises the full interactive surface. Known,
  documented limitation, not a regression target.
- ~~**No Season 2+ mechanics** (Transformations, Overdrive, Astral Heat) — out
  of scope per original brief.~~ **Stale as of Season 5.** Overdrive and
  Astral Heat are both implemented and tested (`test_blazblue.js`); see §4.
  Transformations are still unimplemented and still out of scope.
- **Vega's Innate/Exceed edge-position mechanic: FIXED 2026-08-14.**
  `evalMatchup` now applies `CHARACTERS.vega.specialsUltrasStatModsIfAtEdge`
  to Specials/Ultras via a local `isAtEdgePos`/`vegaEdgeGrant` helper (mirrors
  the `extraCriticalStatsIfOpponentAtRange1` pattern), and `bestMoveTarget`
  takes a narrow Vega-only nudge (`vegaEdgeTargetDistance`) that breaks ties
  toward retreating to an edge when his hand favors Specials/Ultras. Verified:
  `evalMatchup` scores +2 at an edge (+1 more Exceeded, matching the printed
  +3/+1 SPD), confirmed via a direct `node` script. (Scarlet Terror's boost
  text still references an unfinished edge-triggered combo stub — `// HARD
  PART` comment in cards.js — that's a separate, still-open engine gap, not
  an AI gap.)
- **Akuma's headline mechanic: IMPLEMENTED 2026-08-14** (was a missing engine
  feature, not just an AI gap). New `CHARACTERS.akuma.grantsMutualCriticalPow
  (player)` (same function-per-player shape as Zangief's/Vega's extension
  points, but MUTUAL — applies to both `aStats.pow` and `dStats.pow`, not just
  the owner's), consumed in `engine.js`'s `resolveStrikeSteps` right after
  both sides' `liveStats` are computed, gated on it being Akuma's own card
  that paid Critical. `hasRealCriticalValue`/`criticalGrant` in ai.js now
  recognize his 4 no-`critical`-field cards as carrying real value; a new,
  modestly discounted pay probability (`CRITICAL_PAY_PROB_AKUMA_MUTUAL_ONLY =
  0.70`, vs. the normal 0.85) applies only when the mutual grant is a card's
  *sole* source of real Critical value, acknowledging (without a full
  game-theoretic model) that his bonus also helps the opponent. Verified via
  direct `node` scripts: Akuma's own card gains +2 POW (not Exceeded) / +3
  (Exceeded) exactly when his own Critical is paid, in both his structural
  "attacker" and "defender" slots (Speed determines real activation order
  either way); no change when he doesn't pay or when the opponent pays on
  their own unrelated card. Still unimplemented at the engine level
  (pre-existing, unrelated to this fix): Hyakkishu's range-1 immunity
  After-effect, two "Hit: add to Gauge" self-triggers, Demon Armageddon's
  "double all positive POW bonuses," Wrath of the Raging Demon's conditional
  +3 SPD.
- **C. Viper's Continuous-Boost-engine scoring gaps: FIXED 2026-08-14.**
  `evalMatchup` now scores `card.noHitWithoutContinuousBoost` (Burst Time) as
  a guaranteed miss when the setting player has no continuous boost active,
  reads `card.stunImmuneBase` into the same stun-immunity flags boost/Critical
  grants already feed, and credits `powBonusIfOwnContinuousBoost` (Emergency
  Combination) and `opponentPowMod` (Burning Dance's opponent debuff) —
  gated on a new `hasOwnContinuousBoost` flag threaded through
  `boostModsFromKeys`/`engine.activeBoostMods`. `estimateBoostTimingValue`'s
  "boosted" simulation branch now sets that flag true and carries
  `opponentPowMod`, so playing one of her continuous boosts correctly shows
  up as unlocking Burst Time's hit and Emergency Combination's bonus in the
  boost-vs-strike timing comparison. Verified via a direct `node` script:
  Burst Time scores as a guaranteed-miss/wasted-Ultra with no continuous
  boost active vs. a real ~10-POW hit with one active; Emergency Combination
  scores higher with a continuous boost active than without. The pre-existing
  "may Strike after a Continuous Boost" follow-up heuristic in `ui.js`
  (`aiWantsFollowupStrike`) was untouched — it already worked, this fix is
  upstream of it (making the AI value boosting/the Ultra correctly in the
  first place).
- **Audited 2026-08-14 (C. Viper, Vega, Akuma), all three fixed same day** —
  see `CHANGELOG.md` for the full audit-then-fix history, including three
  parallel subagents used to implement the fixes (in isolated git worktrees,
  since they all touched `ai.js`) and the merge process. This project has NOT
  re-audited every remaining character for the same class of AI bug
  (wrong-direction range slack in `effectiveRangeBand`, boost-granted movement
  invisible to the AI, generic mechanic-invisibility like the three above)
  beyond Ryu/Ken/Zangief/Cammy/Vega/Akuma/C. Viper. If a character is
  reported as "playing badly," start by grepping that character's cards for
  `Before: Advance/Close` text and any character-level `CHARACTERS[key]`
  extension fields, then cross-checking each against what `ai.js` actually
  reads, rather than assuming it's covered.

## 7. Blocked engine primitives

Real gaps in engine.js/ui.js, confirmed absent via grep (not assumed), left
as `BLOCKED`/`DEFERRED` comments rather than worked around with invented
fields:

- **Revert (un-Exceeding) — STALE, already false as written.** Re-confirmed
  2026-08-16: `engine.doRevert(player)` exists, is genuinely wired (called
  from `ui.js` at both the human and AI turn-end paths, e.g. Leo's "Revert
  when you end a turn without Striking"), and is a real generic primitive —
  not Leo-specific. The "Exceed-Mode-only cards become unplayable on Revert"
  concern in its own header comment turns out to be moot in practice: this
  engine has no `exceedOnly`/`ultra_exceeded`-style card concept at all (grep
  confirms zero matches) — every character's cards are legal regardless of
  Exceed state, Exceed only changes stat bonuses/mechanics via live
  `owner.exceeded` checks. So Nagoriyuki's Exceed side isn't actually blocked
  on a missing engine primitive — see §4's corrected note for his real
  remaining blocker (missing source data, not missing plumbing).
- **Range-conditional stat bonus applied to ALL attacks, not just Criticals.
  FIXED 2026-08-16.** New `CHARACTERS[key].rangeConditionalStatBonus(player)`
  extension point in `engine.js`'s `liveStats` (mirrors the existing
  Critical-only `extraCriticalStatsIfOpponentAtRange1` precedent, but applied
  unconditionally instead of gated on paying a Critical). Unblocked Potemkin
  (promoted to fully playable).
- **Per-character "Normals gain an on-Hit effect" hook. FIXED 2026-08-16.**
  New `CHARACTERS[key].normalsHitHook` generator (gated on `card.type ===
  'normal'`, dispatched via `yield*` in `engine.js`'s `activateStep` so it can
  pause for a real choice — Testament's "may Push or Pull 1" reuses the
  existing `grasp_move` yield with `min:0` to allow declining). Unblocked
  Testament (promoted to fully playable). Giovanna's narrower need (an extra
  Hit effect on only the ONE Strike her action triggers, not every Normal)
  turned out to need a sibling primitive instead —
  `CHARACTERS[key].actionStrikeHitHook`, gated on the pre-existing
  `actionStrikeThisTurn` flag — also implemented same day.
- **"May Exceed" as a conditional choice inside a hook context** (distinct
  from the existing automatic `onExceed` trigger). Needed by Leo's Innate.
  **FIXED 2026-08-16**, alongside the "pick one of N" fix below: the
  `onAfterStrike` character-level dispatch (`engine.js`, fires after every
  Strike, was a plain function call) is now `yield*`-driven, so Leo's hook is
  a generator that yields a real `may_exceed_choice` (simple yes/no) instead
  of auto-Exceeding unconditionally.
- **Generic "pick one of N named effects" yield type. FIXED 2026-08-16.** New
  `named_choice_effect` yield type (`{options: [{key,label}], allowDecline}`,
  new `named_choice_effect`/`may_exceed_choice` UI modes in ui.js/render.js,
  AI heuristic + `test_headless.js` driver cases for both) — used by
  Millia's Innate/Exceed ("may draw 1, Advance 2, Retreat 2, or gain
  Advantage") via the same now-generator `onAfterStrike` dispatch above.
  Unblocked Millia (promoted to fully playable). Not `name_choice` — that
  existing, differently-shaped mode (naming a card key from a pool, e.g.
  Focus/Block) was left alone to avoid conflating the two.
- **Sagat's face-up Strike mechanic. FIXED 2026-08-16.** New
  `CHARACTERS[key].canGoFaceUp`/`.faceUpStatBonus(player)` + a real
  strike-time choice (`face_up_choice` mode, offered right after picking the
  strike card, before the Critical prompt — human via `doHumanStrike`, AI via
  `aiInitiateStrike` + new `chooseFaceUp` heuristic in ai.js). Sets
  `player._faceUpThisStrike` (per-strike flag, same pattern as
  `_unhittableThisStrike`/`_rangeImmuneThisStrike`), read by `liveStats` for
  the stat bonus. Skipped on a Wild Swing (the engine treats even the owner
  as not knowing that card yet, same reasoning as `maybeAskCritical`'s
  `hideCardName`). Unblocked Sagat (promoted to fully playable); also fixed
  his Exceed's "Retreat up to 2" (auto-resolves to the max, the established
  "up to X" convention — was a separate, smaller TODO on the same character).
  **Known scope limit, documented rather than silently assumed away**: when
  a face-up reveal happens, the human player SEES the real card name
  (`defend_select`'s new `faceUpNote`), but the AI's own defense-selection
  heuristic (`chooseStrikeSelection`) does not specially exploit a
  human-revealed face-up card — it still samples a plausible hand the same
  way it always does. Teaching it to use real information here specifically
  would mean threading a new "known attacker card" input through
  `chooseStrikeSelection`/`evalMatchup`'s Monte Carlo sampling, which no
  other yield in this codebase does yet; deferred as a genuine follow-up, not
  a broken feature — the AI simply doesn't get *better* than blind defense
  from information it's mechanically allowed to have.

## 8. Known-good invariants — don't break these

1. **AI never reads the opponent's actual hand array.** Every AI function
   takes a `publicInfoOf(player)` snapshot, never the live player object.
2. **`resolveStrikeSteps` only yields for genuine per-instance player
   choices, never for something auto-resolvable.** Current yield types:
   `grasp_move` (generalized push/pull, not Grasp-specific — carries
   `min`/`max`/`forceDirection` from the card), `block_force`,
   `hadoken_recycle`, `sustain_choice`, plus several Axl-era yields
   (`optional_gauge_spend`, `optional_retreat`, `decklook_choice`,
   `push_pull_choice`, `move_direction_choice`, `cancel_choice`,
   `critical_choice`), plus `named_choice_effect` (the generic "pick one of N
   named effects", reused by Millia, Chipp, Happy Chaos and Platinum instead of
   adding per-character modes — reach for it first) and `place_eddie` (Zato-1).
   If you un-simplify another card, add a new yield type, but keep the default
   path yield-free.
2b. **A card's Cancel flag lives at `card.boost.cancelable`, never on the card
   itself.** That is the only place anything reads (ui.js ×4, render.js ×1), so a
   top-level `cancelable: true` is inert: the card can never be Canceled and shows
   no Cancelable badge. The entire Season 4 batch did exactly this — 50 cards
   across 14 fighters, silently — until `test_carddata.js` caught it on
   2026-08-17; that test now reports a card-level flag as `INERT_CARD_LEVEL_FLAG`.
   When transcribing, a 'C' in the sheet's Cncl column goes INSIDE the boost object.
3. **Free (card-effect) movement vs. paid (Move-action) movement use
   different pass-cost rules.** Free movement (`advanceSelf` et al.) costs 1
   unit to jump over the opponent's square. The paid Move action
   (`advanceSelfPaid`/`simulateAdvance`) costs 2 Force for that one space.
   Don't unify these — different per the rulebook's Movement section.
4. **Loud discard: any discard path must log the specific card(s) discarded**
   by name — never silent, never count-only. Standing user instruction. Includes
   a Stunned card's forced discard (`resolveStrikeSteps`' second-acting-card-
   stunned branch) — this was the one path found still violating the rule
   (logged only "their attack is discarded", no name) and has been fixed.
4b. **Loud draw (mirrors #4): any card drawn into the human ("You") player's hand
   must log its name** — `engine.draw()` does this centrally for every draw
   (Prepare, Mulligan, `nowDraw`/`cleanup.draw` boost effects, opening hand, etc.)
   via `player.name === 'You'`. Standing user instruction. Deliberately NOT done
   for the CPU/opponent's draws — logging those would leak hidden information the
   no-cheating invariant (#1) is built to protect.
5. **Card cost display: Force (Boost) and Gauge (Ultra Strike) costs must be
   shown on every card tile everywhere cards render, via `render.js`'s
   `gaugeCostBadge()`/`forceCostBadge()`.** Zero cost = no badge shown at
   all, not "[0 Force]".
6. **The Innate/Exceeded action button (`render.js`'s `actionButtons()`)
   must only render when `CHARACTERS[key].action` is truthy.** Pure-passive
   characters (Cammy, Zangief) must show no button.
7. **How to Play's "Reading a card" section must stay in sync with card
   rendering.** Update it and `verify_howtoplay.js`'s assertions whenever
   card-tile structure changes — it's hand-authored, not generated, so it
   can silently drift (has happened once already).
8. **An unconditional self-stat modifier belongs in `liveStats`, not a
   `before` hook.** A `before` hook only runs during that card's own
   `activateStep`, which a FASTER opponent's activation can run first —
   caused a real bug (Sickle Storm's Lose-Armor not applying against a
   faster Ryu). A CONDITIONAL self-stat change (depends on this card's own
   Before-movement outcome) is the one case that still needs hook-based
   mutation; test against a faster opponent specifically if you go this
   route.
9. **Any new movement effect toward/away from the opponent must go through
   `advanceSelf`/`pushToward`/`pushAway`, never raw position arithmetic** —
   easy to reintroduce a same-square collision or missing jump-over bug
   (this happened with the original `pushToward`).
10. **Anything that enumerates "the Normals" for a character must use
    `char.normals || NORMALS`, never the bare `NORMALS` constant** (or the
    dead `NORMAL_KEYS` near the top of ui.js) — there are now two Normals
    pools (`NORMALS`, `GG_NORMALS`), and hardcoding one is invisible until
    tested against a character using the other. Bit three separate places
    (`opponentUniqueKeys`, Focus's naming pool, `deckReferenceHTML`) before
    being caught.
11. **Focus's movement immunity (`player._focusActive` / generalized
    `card.movementImmune`) only blocks the OPPONENT moving you** — never
    blocks your own self-movement (Dive/Tatsumaki/etc.). Any new movement
    effect should route through `pushAway`/`pushToward` (or check
    `target._focusActive`/`movementImmune` itself) or immunity silently
    won't apply.
12. **`new GameController(humanChar, aiChar, difficultyKey, opts)` — that
    order.** `aiChar` is optional at the engine level (random fallback) but
    shell.html always supplies it explicitly. Any stale 2-arg call site breaks
    in a confusing way (difficulty string treated as character key), not a
    clean crash. `opts` (added 2026-08-17, currently just `{beginner:true}`
    from Quick Start) is genuinely optional and defaults to `{}` — do not
    make it required, and do not let beginner mode change any RULE: it may
    only gate prompts that have a no-op answer and change presentation, so a
    beginner-mode game stays a real game the engine can't tell apart.
13. **AI Ultra affordability is filtered using the CALLER's own Gauge
    only** (`chooseStrikeSelection`'s `myGaugeCount`) — never approximate the
    opponent's affordability from their public Gauge count as a substitute
    for filtering the wrong side's candidates.

## 9. Pitfalls (read before debugging anything weird)

- **`let`/`const`/`class` module-level globals do not attach to a Node `vm`
  context object.** `ctx.G = new GameController(...)` from outside a
  `vm.runInContext` call does NOT touch a `let G` binding declared inside
  code run via that call — it silently creates an unrelated property. This
  caused `render()`'s `if (!G) return` guard to fire on every single test
  call for most of a session, with 0 exceptions thrown — "0 crashes" was
  validating engine/controller state only, never actual HTML generation.
  Fixed by using `var G = null;` in ui.js (var/function DO attach to the
  context object; `let`/`const`/`class` never do — classes always need an
  explicit exposer, e.g. `this.GameController=GameController;` appended
  after the bundle). If a new Node-based test "passes" suspiciously easily,
  verify the code under test is actually executing, not just that nothing
  threw — add content assertions, not just crash checks.
- **Movement helper families are not interchangeable.** `advanceSelf`/
  `closeSelf`/`retreatSelf` = free card-effect movement (jump-over costs 1
  unit). `advanceSelfPaid`/`closeSelfPaid`/`retreatSelfPaid` (+
  `simulateAdvance`/`simulateClose`) = paid Move action (jump-over costs 2
  Force), used only by `ui.js`'s `confirmMove()`. `pushAway` = moves target
  away from source, can't collide. `pushToward` = moves target toward
  source, delegates to `advanceSelf` for correct jump-over handling (was
  buggy raw arithmetic before a fix).
- **A probability-gated heuristic branch must re-check the SAME
  precondition an earlier branch already checked**, not just its own. The
  original `chooseAction()` had a later unconditional `if (rng()<0.55)
  return {type:'strike'}` that fired even with zero cards in range, because
  it didn't re-gate on `inRangeCount===0` the way an earlier branch did. No
  crash, just quietly bad play. When editing `chooseAction()`, trace every
  `rng() <` branch back to its implicit precondition.
- **A function signature with unused parameters can silently no-op an
  entire class of behavior.** `evalMatchup`'s `myArmMods`/`oppArmMods`
  parameters existed but were never read by the function body for a long
  time — every caller passed `null`, and boosts were invisible to strike
  scoring. Nothing crashed; the AI was just quietly worse. When adding a new
  stat-modifying mechanic to the engine, check whether `evalMatchup` needs
  to know about it.
- **A function with a comment describing intended behavior but an empty
  body is a bug, not a design decision.** `engine.moveChoice()` had exactly
  this shape (comment said the caller would "typically" handle it; no caller
  did) and silently no-op'd two cards' effects for the project's entire
  life. Grep for short/empty method bodies with suspicious comments if a
  card's text doesn't seem to be happening.
- **The bundler's `module.exports` stripping is regex-based and assumes the
  export block is the file's last statement** (see §2) — code after it gets
  silently deleted. `node -c bundle.js` is the check that catches a malformed
  strip, run it every time, not just when something looks wrong.
- **Round-log numbering**: `engine.roundNum` increments both inside
  `resolveStrikeSteps` and via `engine.startRound()` (called at the top of
  every non-strike action handler). If you add a new non-strike action path,
  call `engine.startRound()` at its top or it won't be numbered/grouped in
  the log. If a `startRound()` call is followed by a failure path where
  nothing happened, pop the header back off
  (`engine.log.pop(); engine.roundNum--;`) to avoid an empty round-card.
- **"Goes to Gauge after N turns" boosts (`expireToGaugeAfterTurns`) must
  not count the turn they were played on.** `playBoost()` sets
  `entry._skipNextExpiryCheck = true` at creation; `checkBoostExpiry()`
  consumes that flag on its first call without decrementing, so the real
  countdown starts from the player's genuinely next non-strike turn. Without
  this, `1 - 1 = 0` fires in the same synchronous continuation the boost was
  played in (was a real, shipped bug — Cannonball/CQC never got even a
  moment of effect before the fix).
- **Difficulty ladder noise**: `AI_DIFFICULTIES` (champion=0, warrior=0.20,
  student=0.40 randomness) shows a clean monotonic trend vs. a fixed weak
  baseline and in two-tier-apart matchups, but **adjacent-tier head-to-head
  (Champion vs Warrior) lands near 50/50 even at N=1500** — real in-game
  luck dilutes a 20-point decision-quality gap over one matchup. This is
  expected, not a bug; don't "fix" it by raising Warrior's randomness past
  spec. If the ladder check ever fails after an unrelated change, first
  check whether the fixed "dumb" baseline is silently picking up a new
  optional-decision benefit (this happened once with Critical payment
  before "dumb" was explicitly excluded from it) before assuming a real
  regression.
- **`this.gameId`, `engine.pullToward`, `engine.movePlayer`, `engine.doCharacterAction`
  do not exist — do not reinvent them.** Real methods: `pushToward(target,
  source, amount)` (pull, may cross), `pullTowardWithoutPassing(target, source,
  amount)` (pull, stops adjacent), `advanceSelf`/`retreatSelf` for self-movement,
  and for character actions specifically:
  `CHARACTERS[key].action.resolve(engine, player, opponent, direction)` (see
  `ui.js`'s human-side `runCharacterAction()` for the canonical call shape — the
  AI-side `runAITurn()` diverged from it and called a nonexistent
  `engine.doCharacterAction(...)` instead, which crashed ~1/50 games in
  `test_headless.js` until caught and fixed). These four invented names were
  each written and shipped once, then caught only when a test-coverage gap was
  fixed (or, for `doCharacterAction`, when Node finally became available again
  to actually run the existing suite) and the code path finally executed for
  the first time (see next pitfall).
- **A green test suite only proves what the test suite's driver actually
  calls.** `test_headless.js`'s random action-picker never included
  `actionSpecial()` for a long stretch of the project's history — meaning
  EVERY character-action code path (any `CHARACTERS[key].action`) had a 0%
  chance of executing in any automated test, ever, despite "0 crashes across
  2000+ games" being reported as true. Fixing the driver gap immediately
  surfaced the three invented-API bugs above and an 11-character missing-
  `normals:` data bug, none of which had ever run before. When adding a new
  yield type, hook, or `G.mode`, verify with a targeted grep (e.g. `grep -oP
  "g\.\w+" test_headless.js | sort -u`) that some driver path actually
  reaches the new method — a passing suite does not imply this on its own.
- **`resolveStrikeCardSelection` can legitimately return `null`** (true
  deck+discard exhaustion — it sets `gameOver`/`winner` itself). Every
  caller must null-check before reading `.card` off the result. Two AI-side
  call sites in ui.js (`aiChooseDefense`, `aiInitiateStrike`) were missing
  this guard for a long time — invisible until a character with aggressive
  discard effects (Zangief) made exhaustion reachable in practice.
- **Copyrighted third-party art/sprites are off-limits regardless of "the
  wiki has them publicly."** Declined requests to source Capcom sprite art;
  built original geometric silhouettes instead, later swapped for
  user-supplied portrait files already sitting in the project (a materially
  different situation — supplied by the user, not fetched externally on
  request). If a similar request comes in for an asset not already in the
  project, treat it as the original declined case, not the swap case.

## 10. If the user reports "the game doesn't work" after a change

1. Rebuild the bundle (§2) and run `node -c bundle.js` — malformed
   exports-stripping is the most common self-inflicted failure.
2. Run the test suite (§5). `Unhandled mode` in `test_headless.js` means you
   added a UI mode without a driver case — add one, don't delete the check.
3. If tests pass but the user still sees a problem, it's likely something
   the headless DOM stub can't see (rendering fidelity, CSS, touch events) —
   ask for a screenshot or open the file in a real browser.
4. Re-read §8 (invariants) before changing engine.js/ai.js — easy to
   violate the no-cheating guarantee or a yield-type constraint while
   "simplifying" code.
