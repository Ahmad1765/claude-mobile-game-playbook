# Mosaic Parade: Development Log

Game 04 of the portfolio (spec: `../games/04-pixel-parade.md`). A conveyor "colour shooter" in the style of Pixel Flow: a pixel picture sits inside a belt; shooters ride the belt and fire at tiles of their own colour; pick the order, don't jam the belt. Every finished picture goes into an album (10 per page), hard levels every 5th from 15, a daily picture, an "extra slot" booster. Ads and other monetization are deliberately not built yet. Test device: Infinix X655C (Helio P22, Android 10, system font scale 1.15). Started 2026-10-20; desktop-playable and device-tested v1 the same day (regression 9/9, 60 fps).

## Day-one decisions (Gemstack retrospective §5.1)
| Decision | Value | Why |
|---|---|---|
| Name | **Mosaic Parade: Color Shooter** | The spec's "Pixel Parade" is taken on Google Play (an app of exactly that name). Also taken: **Cube Parade: Sort Puzzle**, **Pixel Conveyor**. Free on Play and the web (2026-10-20): Mosaic Parade, Pixel Promenade, Color Caravan, Pixel Carousel (the last too close to the iOS game *Color Carousel*). Chosen by the owner |
| Package id | `app.mosaicparade.game` (**final**) | Same per-game pattern as the earlier games |
| Upload key | not made yet | Make `android/mosaicparade-upload.jks` + `keystore.properties` before the first release, as for Arrow Garden |
| Perf budget | 60 fps idle, ≥55 fps while the belt runs, on the P22 | Met: see Measurements |
| Stack | Capacitor 8 + Canvas 2D (copied from Arrow Garden) | The picture is static; only up to 5 tokens and the shots move |

## What was reused (the reuse-first rule)
Copied `../arrowgarden/` whole (minus build outputs, keys and node_modules) and trimmed: the Android project with the edge-to-edge `MainActivity` (package renamed), `store.js`, `dom.js`, `fx.js`, `icons.js`, the card/button/modal kit, the daily calendar, the level map layout (flower beds became album pages), the deferred-save and win-recorded-at-once patterns, the per-mode resume slots, `tools/device.mjs`, `shot.mjs`, `release.mjs` (with the empty-file guard), `regress.mjs`'s harness, `make-art.sh`. The Android debug build passed on the first try for the third game running.

New code: `pictures.js` (30 hand-drawn pictures), `logic.js` (belt simulation, greedy solver, queue generator, schedule), the renderer and docks in `game.js`, the album in `app.js`, new art in `sheet.html`.

## How it works
- **The belt** (`logic.js`, pure, tested) is a loop of stops round the picture: one per lane plus one per corner, starting at the bottom-left corner, clockwise. At each lane a shooter looks along that row or column and fires at the first tile, **only if it is its colour** (one tile per lane per pass: `BURST = 1`). Out of ammo, it leaves at once. Back at the start with ammo left, it parks in a slot; **no free slot = the belt jams** (the only way to lose). Shooters enter 3 stops apart (a token is about 3 tiles wide); the belt holds `cap` at a time.
- **One simulation for everything.** The game, the solver, the tests and the `__dbg` bot all call `newState` / `send` / `tick`; `tick` returns events (enter, shot, gone, parked, jam, win) that the renderer animates. Ammo of each colour always adds up to its tile count, so a win empties every queue and slot.
- **Levels.** Each art pixel becomes 2×2 tiles (level 1 stays 1×1), so pictures are 14–30 tiles across while the art stays simple. Every picture has a **background colour**: a full square buries the inner colours, which is where the puzzle comes from. Shooters are cut from each colour (5–40 ammo), ordered by when their tiles surface (peel depth) plus noise, and dealt round-robin into 2–3 queues. Of up to 6 candidate line-ups, the most shuffled one the greedy solver wins **with a spare slot** is kept; if none, the level gets the slots its solution needs. Hard levels (every 5th from 15) have one slot fewer and no spare.
- **The greedy player** (`pick`) sends the shooter whose colour fronts the most lanes, slots first on a tie. It is the solver, the tutorial's glow, and the regression's bot.
- **Rendering** (`game.js`). The picture and belt are one static canvas; a shot repaints one cell (a faint "ghost" of the tile stays, so the picture's outline remains). Shooters are DOM tokens moved by `transform`; shots are pooled DOM beads animated by WAAPI; queues and slots are DOM buttons re-rendered only when they change. The tick loop runs only while something is on the belt or landing. A lap takes about 4 s whatever the picture's size.
- **Meta** (`app.js`). Home shows the next picture; the album (one page per 10 levels, finished pictures shown in place of their numbers, hard levels ringed); the daily with a month calendar; settings; the Android back button. Extra slots start at 3; +2 per finished page, +1 per first daily. The booster can be used any time or on the jam card ("Add a slot and keep going"). Stars: most shooters waiting at once 0–1 = 3, never all slots = 2, all slots or a booster = 1.

## Balancing, measured (`npm test`, 300 levels)
| Levels | Tiles | Shooters | Greedy peak slots | Greedy ticks | Random player wins |
|---|---|---|---|---|---|
| 1 | 42 | 8 | 0 | 47 | 100% |
| 2–5 | 287 | 23.8 | 1.5 | 320 | 100% |
| 6–10 | 361 | 22.4 | 1.8 | 487 | 100% |
| 11–20 | 391 | 20.1 | 2.3 | 579 | 80% |
| 21–30 | 512 | 25.3 | 2.5 | 939 | 65% |
| 31–60 | 405 | 20.9 | 2.3 | 721 | 70% |
| 61–100 | 405 | 20.4 | 2.3 | 737 | 27% |

"Random player" picks any queue front or slot at random every ~250 ms (2 seeds per level, levels 1–100). It is the difficulty measure: how much the choices matter. Ticks are 32–90 ms each (a lap takes about 4 s). Generation: 4.0 ms average, 14 ms worst on desktop; 23–54 ms on the P22 (see Measurements).

**How the balance was found** (each step measured with the random player, levels 11–40):
1. First version: burst 3, 1×1 art, transparent backgrounds. A level lasted one lap (19–53 ticks) and the greedy player never parked a shooter.
2. Burst 1 and 2×2 tiles: levels took 190–350 ticks, but a random player still won 99%, a careless one (always queue 1 first) 90–100%.
3. Bigger ammo, smaller belt, more shuffle: still 80–100%. The cause wasn't the knobs: **nearly every colour touched the picture's edge**, so almost any shooter was useful at any time.
4. **Background colours** (full squares, like the genre): random wins fell to 83–100%.
5. **Slots are the knob that makes choices matter**: 3 slots → random wins 58–65%, 4 → 80–85%, 5 → 93%+; the greedy player wins all of them. The schedule now goes from 5 slots to 4 and a 3-shooter belt.

## Measurements (on the phone, font scale 1.15)
| Scenario | Result |
|---|---|
| Level generation + layout (`__dbg.start`) on levels 12, 22, 24, 48, 120, 144 | 25, 47, 45, 54, 23, 36 ms |
| Idle, level 22 | 60 fps, 0% late frames |
| A full belt: 4 shooters firing at once | 59.8 fps, 0.3% late (one 33 ms frame at the sends) |
| Greedy play for 7 s, level 30 | 59.9 fps, 0.2% late |

DOM tokens moved by `transform` and WAAPI beads cost nothing measurable, even with 4 shooters firing every tick. Every screen checked at 1.15 and 1.3 text: nothing truncated (the jam card's button wrapped to three lines at 1.3 and was shortened).

## Bugs found by looking and testing
- **A shooter's last shot crashed the renderer's lookup**: it leaves the belt in the same tick it fires, so the renderer couldn't find it. Events now carry the shooter itself.
- **`begin(P, run)` shadowed the `run()` loop function**, so resuming a level with shooters on the belt threw "run is not a function". Caught by the desktop smoke test's page-error check (the §5.12 lesson again).
- **A regex lost its backslash** through a Node patch script, so `/./g` painted every tile with the background (level 1 had 42 tiles of one colour). The test table showed it at once; replaced with `replaceAll('.', bg)`.
- **Faded "next in line" tokens read as other colours** (half-transparent brown looks beige, a real shooter colour). Next-in-line tokens are now smaller but full colour.
- **White tiles on a cream card became graph paper.** The picture sits on a grey backing board; no per-tile outlines.
- **The belt touched the screen edge** on 28-tile-wide pictures: the layout margin now includes the belt band plus an 8 px gutter.
- Generation was 60 ms worst on desktop after the balancing changes (about 300 ms on the P22). The solver re-scored every lane every tick; it now scores only after a tick with events, and each lane's front tile and the per-colour counts are kept incrementally. Stats unchanged (same levels), 25 ms worst.
- **The tutorial glow was missing at the start** of level 1 (it was only applied after the first event). Caught by the device regression's first assertion; desktop screenshots had not shown it because nobody looked for the glow.
- **Progress was only saved on backgrounding or quitting.** `saveRun()` cleared the pending timer but did not reset its id, so `saveSoon()` believed a save was still pending and never scheduled one again. Caught by the regression's daily-resume check (a resumed daily came back fresh). A kill without backgrounding would have lost the whole level.
- **Album page names repeated** after the 30th picture (level 31 was "Sunny days" again). Repeat pages are numbered: "Sunny days 2".
- **Empty stray files** again: 16 at the project root this session (`(boosted`, `0)`, `G`, `save`, `{` …), none in `www/`. Deleted; `release.mjs` still guards `www/`.

## Lessons for the next game
**What worked (keep doing it)**
- **Copying the last game whole, a third time.** The Android build passed first try; only names, the package and art changed outside the game code. Game 05 should copy this folder.
- **A random player's win rate as the difficulty number.** "The solver wins" only proves a level is fair; "a random player wins 99%" proved it was pointless. The test prints both per band.
- **The same stats table before and after a speed-up** is a free equivalence test: two optimizations left every number identical.
- **Writing the device regression and a desktop smoke script while the phone was unplugged.** The smoke script caught one crash; the regression caught two more bugs within minutes of the phone coming back.

**What to watch for**
- **Look for the structural cause before sweeping knobs.** Ammo, belt size and shuffle hardly moved the random win rate; the cause was that every colour touched the picture's edge.
- **In a colour-matching game, never fade a colour** to mean "inactive": it reads as a different colour.
- **adb screencap lags about a second.** To catch something fast-moving, capture in parallel with the action, and check the DOM before concluding that it doesn't render.
- **Difficulty after level 60 is only tuned by the random player.** Play it by hand.

## Commands
| Task | Command |
|---|---|
| Logic tests (content, rules, 300 levels, 400 dailies, random-player difficulty, timing, stats table) | `npm test` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start(18)"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Signed Play bundle | `JAVA_HOME=… npm run release` (needs the upload key first) |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |

adb is not on the PATH: `$LOCALAPPDATA/Android/Sdk/platform-tools/adb.exe`.

## Still to do
- Play it by hand: lap speed, level length (the greedy bot takes ~20–40 s; people will be slower), whether the 2×2-tile pictures read well at 10–13 px tiles.

## Not done yet (by decision)
- **Monetization:** ads, a rewarded "continue" (+1 slot) on the jam card, rewarded boosters, interstitials. The extra-slot economy is the hook.
- Coins and the spec's other boosters (shuffle queues, magnet, bomb). Add if players get stuck in ways an extra slot doesn't fix.
- Weekly themed events, chests for finished pages. New pictures are cheap (append to `pictures.js`), so events are mostly content.
- Win celebration effects (confetti): the picture card is the celebration for now.
- Store screenshots, hosting `store/privacy.html`, the upload key.
- **Generator changes change levels.** Saved stars stay; a saved run whose picture no longer matches restarts fresh (guarded in `restore()`). Freeze the generator and `pictures.js` order once it ships (pictures are append-only).
