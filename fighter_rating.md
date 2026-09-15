# Fighter Rating

Ratings for every playable fighter in this Exceed implementation, on two axes:

- **Complexity** — how heavy the fighter's mechanics are: how many forced decision
  points they add per turn, how much persistent state the player must track, and how
  far they deviate from the plain move / strike / prepare / change-cards baseline.
- **Win Rate** — how often they actually win.

Each axis is rated twice: **stars** (1–5, half-stars possible) and an **order**
(1 = highest, out of 41).

**Roster size:** this file covers **41** fighters — every key in `CHARACTERS`
(`cards.js:5109`), which is exactly what `rosterKeys()` (`ui.js:19`) treats as playable.

**Basis:** all judgements below come from the mechanics as *implemented* — `cards.js`
character entries and card definitions, `engine.js` (turn loop, `resolveStrikeSteps`,
stat resolution), and the `G.mode` prompts in `ui.js` that a human player is actually
asked about. Printed-but-deferred card text is treated as absent, because it is.

---

## 1. Complexity

Scale anchor: **1.0** = vanilla-shaped specials and a passive that never asks you
anything. **5.0** = a fighter that demands constant tracking and several branching
choices every single turn.

| # | Fighter | Stars | Score | Signature mechanic |
|---|---|---|---|---|
| 1 | Zato-1 | ★★★★★ | 5.0 | Eddie — a third board token every Special measures Range from |
| 2 | Happy Chaos | ★★★★½ | 4.5 | A sealed-card zone that eats his own deck as ammunition |
| 3 | C. Viper | ★★★★ | 4.0 | Optional bonus Strike after *every* Continuous Boost |
| 4 | Platinum | ★★★★ | 4.0 | Reveal-and-play a free boost each exchange; a whole boost board to maintain |
| 5 | Nagoriyuki | ★★★★ | 4.0 | Reshuffle force-Exceeds him and permanently bricks all seven Specials |
| 6 | Anji Mito | ★★★½ | 3.5 | Buy Guard with Force on every attack you set, attacking or defending |
| 7 | ν-No. 13 | ★★★½ | 3.5 | Rent +4 Range at set time — on offence *and* defence |
| 8 | Hakumen | ★★★½ | 3.5 | Gauge is both the exceed clock (cost 8) and every Special's mid-Strike fuel |
| 9 | Bang | ★★★½ | 3.5 | Deck-sifting Action plus Overdrive that rakes back every copy of a card |
| 10 | Jack-O | ★★★½ | 3.5 | Boosts-in-play are a live counter her cards scale off |
| 11 | Faust | ★★★½ | 3.5 | Every Action force-boosts a random card off his own deck |
| 12 | Cammy | ★★★½ | 3.5 | Side-swapping guarantees range; stacked range-immunity windows |
| 13 | Baiken | ★★★ | 3.0 | Four cards check a live "3 or fewer cards in hand" threshold |
| 14 | Tager | ★★★ | 3.0 | Hard cap: he can never self-move more than 2 |
| 15 | Jin | ★★★ | 3.0 | Pay Gauge at set time for Draw 2; delayed boosts and hand-size Guard |
| 16 | M. Bison | ★★★ | 3.0 | Bespoke multi-select hand-to-Gauge Action |
| 17 | Axl | ★★★ | 3.0 | +2 Range welded onto the whole Normals pool, plus four paid prompts |
| 18 | Testament | ★★★ | 3.0 | A free reposition prompt on every Normal that connects |
| 19 | Chun-Li | ★★★ | 3.0 | Retreat prompt after every Boost; a range-less Head Stomp |
| 20 | Vega | ★★★ | 3.0 | Edge-of-arena zoning his own cards keep walking him out of |
| 21 | Zangief | ★★★ | 3.0 | Character-level bonus stapled onto every Critical paid at Range 1 |
| 22 | Sol Badguy | ★★½ | 2.5 | Cancel-chaining into +POW — Gauge split between Cancels and Ultras |
| 23 | Millia | ★★½ | 2.5 | Stunning the opponent opens a four-option menu |
| 24 | Chipp | ★★½ | 2.5 | An unconditional reposition prompt after *every* exchange |
| 25 | Leo | ★★½ | 2.5 | Free Exceed on stun — then oscillating in and out of Exceed |
| 26 | Ramlethal | ★★½ | 2.5 | Pay {2} at set time to stretch any attack one space further |
| 27 | Noel | ★★½ | 2.5 | Buy initiative: Gauge for +1 SPD at set time |
| 28 | Taokaka | ★★½ | 2.5 | Two optional Force-for-POW spends and a mandatory lunge |
| 29 | Goldlewis | ★★½ | 2.5 | Ten distinct cards and a 36-card deck — breadth, not depth |
| 30 | Akuma | ★★½ | 2.5 | Symmetric Critical: +POW to *both* players |
| 31 | Sagat | ★★½ | 2.5 | Optional face-up Strike — reveal your attack to buy initiative |
| 32 | Guile | ★★½ | 2.5 | Change Cards as an engine: first Gauge spent generates {2} extra |
| 33 | Dan | ★★½ | 2.5 | Self-destructive gambling on random discards and deck rolls |
| 34 | I-No | ★★ | 2.0 | {1} Action: Advance 2 and Strike |
| 35 | Ky Kiske | ★★ | 2.0 | The first Cancel each turn is free tempo |
| 36 | May | ★★ | 2.0 | One Force, always, for +POW when initiating |
| 37 | Ken | ★★ | 2.0 | Close-and-draw every turn, with initiator-conditional burst |
| 38 | Ragna | ★★ | 2.0 | Lifesteal on literally every hit |
| 39 | Ryu | ★½ | 1.5 | Textbook shoto — the baseline the file is written against |
| 40 | Potemkin | ★½ | 1.5 | A flat, promptless +POW/+ARM tax for standing at Range 1 |
| 41 | Giovanna | ★½ | 1.5 | An Action that is simply "Strike, and draw on hit" |

### Notes on the complexity extremes

**Zato-1 (5.0)** is the only fighter with an extra board entity: `eddiePos`
(`engine.js:120`), placed via a `place_eddie` prompt on every Normal hit
(`cards.js:6284` → `ui.js:1297`), with `attackSourcePos` (`engine.js:704`) rerouting
Special range origin to Eddie and `onStrikeCleanup` (`cards.js:6315`) removing him
after most Specials. The plant-with-Normals / spend-with-Specials cycle is a second
game running alongside the first.

**Happy Chaos (4.5)** carries a `sealedCards` zone (`engine.js:80`) fed by an on-hit
`named_choice_effect` yield (`cards.js:6232`), three cards that read the pile back,
Sagat-style face-up initiation, and an Exceed that immediately Reverts.

**Giovanna, Potemkin, Ryu (1.5)** each add essentially one thing to the baseline and
never prompt: Giovanna's `resolve` is a bare `{ triggersStrike: true }`
(`cards.js:6121`), Potemkin is a single `rangeConditionalStatBonus` (`cards.js:5524`),
and Ryu has exactly one interactive hook in his whole kit (`cards.js:361`).
