# Tile Picnic: Triple Match (game 10)

Spec: `games/10-tile-trio.md`. **A fresh project** (`npm init`, `npx cap init`, `npx cap add android`), every file written new, per the owner's no-copy rule (root DEVLOG §5.19). Ideas taken from earlier games: the insets bridge, the device tool and regression style, and from Lotus Mahjong (game 01) the stacked-tile board drawn on one canvas with flying tiles as small canvases, rewritten for this game.

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Tile Picnic: Triple Match** | The spec's "Tile Trio" is taken (**Tile Trio: Triple match - 3D**, Freeplay). Free (2026-10-21): Tile Picnic, Sticker Pantry, Tile Larder, Tile Tea Party, Sketchbook Tiles. Chosen by the owner |
| Package id | `app.tilepicnic.game` (**final**) | |
| Upload key | not made yet | Make `android/tilepicnic-upload.jks` + `keystore.properties` before release |
| Style | **Illustrative / Hand-Drawn** (from the owner's list): a sketchbook page. Cream dotted paper, ink outlines that are never ruler-straight (uneven border radii; tiles traced with a seeded wobble), watercolour washes, cards set at slight angles, a red gingham picnic blanket for the tray, Patrick Hand for all text | Food stickers on a picnic blanket; unlike every earlier game |
| Board | One canvas, redrawn after each move (1.0 ms for 60 tiles on the P22); tray and flying tiles are small canvases moved by `transform` | See Measurements: the DOM version could not hold 58 fps |
| Levels | Pre-built in `www/levels.js`; the daily market and duels are built on the phone (2-4 ms a board) | |

## Outside resources used
| Resource | Licence | Use |
|---|---|---|
| **Microsoft Fluent Emoji**, flat style: 50 food pictures (`www/img/`; there is no pumpkin) | MIT (`www/img/FLUENT-EMOJI-LICENSE.txt`) | Every tile, the recipe strip, the cards, the market row |
| **Kenney** Interface Sounds (pluck, drop, confirmation, bong) and RPG Audio (book flip, cloth) | CC0 | Tap, into the basket, a pop, a full basket, a recipe card, a peek |
| **Fontsource**: Patrick Hand | OFL | All text |
| **Capacitor** app, haptics (official) | MIT | Back button, haptics |

## Selling points (approved by the owner 2026-10-21, all four)
- **Recipe cards.** Every level cooks one of 36 simple, real recipes (written for this game); its 3-4 ingredients are among the level's tiles. A strip under the title shows them and ticks each one when its triple pops. The win card is an index card (ruled, a red margin line) with the ingredients and steps, and it goes into the recipe box.
- **Peek.** Press and hold a covered tile: the tiles on top of it (and on top of those) fade to 14% for 1.4 s. Three per level. Only a covered tile peeks; a slow press on a free tile still takes it (see Bugs).
- **The seasonal market.** A new board every day from the produce in season this month (a broad northern temperate guide written for the game; a setting shifts it six months for the southern half). Built on the phone from the date as seed.
- **Duel.** Two players, one phone: turns alternate after every tile, a triple scores for whoever made it, and whoever fills the basket loses (otherwise most triples wins). No helpers in a duel, and their row keeps its space (`visibility`) so the board never jumps.

## How it works
- **Rules** (`logic.js`, pure, tested): a tile is free when no tile on a higher layer overlaps it. A tapped tile goes into the tray next to its kind; three alike pop; seven in the tray without a pop is a loss. Undo (3 a level) and "put back 3" (1 a level) also work from the loss card.
- **Solvable by construction** (`makeBoard`): pick a legal removal order (always a free tile), then hand out pictures in threes along that order so that following it never holds more than k tiles in the tray (k 3-5, under 7). The rule that makes it true: at every step, the tray's peak if all open groups were finished from there (fullest first) must stay within k; a new group may open only if that still holds. (The first version only capped the current load and lost on its own order: four open groups of one tile each climb to seven.) The order ships with the level, and the test replays it.
- **Layouts** (`makeLayout`): mounds. Each layer fills from the middle outwards with a little noise, mirrored left to right, smaller than the layer below, odd layers half a tile over, and every upper tile resting on one below (tested).
- **Difficulty**: a "decent" player (completes a triple when it can, else adds to a pair while there is room, else any free tile) plays each board 40 times; each slot has a band of win rates, every 5th level from 10 on lower. Measured on the shipped file: levels 1-5 87%, 6-30 56%, 31-80 38%, 81-150 18%, 151-200 10%. (A careless player that only sometimes matches won 0-1% of mid-size boards, too weak to measure with.) 200 levels in 2 s.

## Measurements (on the phone, font scale 1.15)
| What | Result |
|---|---|
| Idle board (level 190, 60 tiles) | 60.6 fps |
| 12 taps 280 ms apart with pops, levels 190, 190, 30, 60 | 59.1, 59.1, 59.8, 59.6 fps |
| Drawing the whole board canvas | 1.0 ms |
| 1.3× text | every screen fits after one fix ("Put back 3" wrapped) |
| Device regression | 7/7, six runs clean after the last fix (`node tools/regress.mjs`): level 1 with real adb taps and its recipe card saved at once; a covered tile refused, a real long press peeking, a slow press on a free tile taking it; a full basket, undo out of it, put back 3; resume; the market; a duel; back button |

### How the frame rate was found (the long way)
The first board used one DOM element per tile: a burst of taps ran at 55 fps with 100-200 ms frames. Moving the board to a canvas (1 ms to redraw), building the recipe strip once per level instead of per tap (new SVG `<img>`s were decoded on every tap), deferring saves off the tap frame (root §5.4, which I had broken), and making flying tiles canvases instead of data-URL backgrounds each changed little (54-57 fps). Switching parts off one at a time pointed at "a tile starts to fly"; **a Chrome trace over DevTools (`tools/trace.mjs`) showed the time in GPU raster, 650 ms in 54 tasks (worst 150 ms)**: tiles moving over the page repainted the full-screen dotted-paper gradient and the gingham tray gradients. Putting each static background on its own layer (`transform: translateZ(0)`, rastered once) gave 59.1-59.8 fps. The other changes stayed (each is cheaper anyway), but the gradients were the cause.

## Bugs found by looking and testing
- **A slow press on a free tile did nothing**: a press over 420 ms was taken as a peek, and peeking only applies to covered tiles. Found by a flaky regression run (adb sometimes delivers a slow tap under load); a hold now swallows the tap only when it really peeks, and the regression checks a 700 ms press.
- **Taps before the pictures were drawn crashed** (the tray reached for a picture still being drawn); found by the regression starting levels back to back.
- **The generator's tray bound was wrong** (above): its own order lost. Found by replaying the order in the measuring script.
- **Upper tiles floated on nothing**: the layout's mirror twin was placed without checking support. Found by the test.
- **The first layouts were scattered**, not mounds: found by looking at the screenshot.
- **The careless player picked its candidate list twice** with different random draws, so its index could fall outside the list.
- **The peek screenshot missed the peek**: `adb screencap` lags about a second (root §5.14); a test hook peeks at the right moment.
- **Heredocs with apostrophes in comments failed** again (root §5.10); those files went through the Write tool.

## Commands
| Task | Command |
|---|---|
| Logic tests (rules, all 200 levels replayed to a clear board, whole triples, recipes present, every month's market) | `npm test` |
| Rebuild the levels (append-only once shipped) | `npm run levels` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.level(40)"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| A trace of where frame time goes | `node tools/trace.mjs "<js>" [ms]` |
| Device regression (after every change) | `node tools/regress.mjs` |
| Signed bundle | `node tools/release.mjs` (needs `android/keystore.properties`) |

## Not done yet (by decision)
- **Monetization**: rewarded extra undos, peeks and basket space; interstitials. Helpers are free and limited per level for now.
- **Golden-tile sticker album** and **themed tile sets** from the spec (the recipe box takes the collecting role).
- **A local seasonal guide**: the market is a broad northern temperate guide with a six-month shift for the south, not a per-country calendar.
- **The level file is append-only once shipped**: saves point at level numbers and tap orders.
