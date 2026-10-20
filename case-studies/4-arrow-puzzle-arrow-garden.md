# Arrow Garden: Development Log

Game 03 of the portfolio (spec: `../games/03-arrow-out.md`). An arrow "tap away" puzzle: every arrow is a path of cells, tapping slides it out head first along its own path, and it bumps if anything sits between its point and the edge. Every 10 levels a flower bed blooms (a flower card with a short fact), picture boards every 5th level, a daily puzzle, 3 hearts, hints. Ads and other monetization are deliberately not built yet. Test device: Infinix X655C (Helio P22, Android 10, WebView 154, system font scale 1.15). Started and reached a playable, device-tested v1 on 2026-10-20.

## v2 (2026-10-20): claymorphism, living seasons, watering, free misreads
After the owner's feedback (root DEVLOG §5.16), restyled from scratch and given three approved selling points.
- **Style: claymorphism.** Soft, puffy, matte clay pieces: an inner highlight at the top-left, an inner rim at the bottom-right and a soft drop shadow, all static (no animated blur). Pill buttons, round icon buttons, Fredoka (OFL) type. On the board, every arrow is a clay tube (a shadow below, the body, a highlight along the top) and every cell a dimple, all drawn only when the board changes.
- **Living seasons.** `seasons.js` picks the season from the date (northern-hemisphere months for now): the page and card colours, the home scene (blossom, sun, falling leaves, snow) and the win particles (petals, sparks, leaves, snowflakes) all change with it.
- **Water your garden.** The home scene shows one clay flower per finished bed; they droop after 2 days without play and wilt after 4 (calendar days), with a line saying so. Any finished level waters the whole garden ("Your thirsty garden has been watered"). Plants never die.
- **Free misreads.** Press and hold an arrow (320 ms) to see where it would go: green dots to the edge, or red up to the blocker, with the blocking arrow outlined. Lifting moves nothing; a quick tap still sends the arrow (on lift now, not on touch). The tutorial and the "Out of hearts" card mention it.
- **Outside resources:** Kenney "UI Audio" and "Impact Sounds" (CC0): a click, a soft roll for arrows leaving, soft clay thuds for bumps, a water splash. The climbing pentatonic notes stay (they are the game's signature). Fredoka via Fontsource.
- **Measured on the phone (v2):** idle 60 fps; arrows leaving 59.3 fps (worst frame 33 ms); a preview 58.6 fps. Every screen fits at 1.3× text. Regression 11/11 (a real long press previews and moves nothing; a thirsty garden is watered by a win).
- **Found and fixed (v2):** the home scene was letterboxed with pale bands (fixed with a taller drawing on a sky-coloured box; `slice` cropped flowers off the sides on tall boxes). The regression's long press first checked too early: adb's touch lands about 0.9 s after the command, so the test now waits for the preview to appear.

## Day-one decisions (Gemstack retrospective §5.1)
| Decision | Value | Why |
|---|---|---|
| Name | **Arrow Garden: Tap Away Puzzle** | The spec's "Arrow Out" is taken several times over: **Arrow Out: Tap Away Puzzle**, **Arrow Out 3D: Tap Away**, **Arrow Out Puzzle**, **Arrows Out**. Also checked and found taken: Arrow Drift (Arrow Drift Escape), Arrowscape(s). No "Arrow Garden" app found (2026-10-20). Chosen by the owner from three free candidates; the garden gives the meta loop a theme and stands apart from the black-and-white minimal clones |
| Package id | `app.arrowgarden.game` (**final**) | Same per-game namespace pattern as the earlier games |
| Upload key | `android/arrowgarden-upload.jks` + `android/keystore.properties` (gitignored) | Own key per game. **Back both up off this machine** and enrol in Play App Signing |
| Perf budget | 60 fps idle, ≥55 fps during effects, on the P22 | Measured below |
| Stack | Capacitor 8 + Canvas 2D (copied from Word Passport) | Static board, one moving arrow at a time: the easiest case for this phone |
| Board size cap | 9 columns × 11 rows, cells ≥ 36 px | Tap targets. The hit games use 15×15+ boards with pinch-zoom; zoom is not built (see "Not done") |

## What was reused (the reuse-first rule)
Copied `../wordpassport/` whole (minus build outputs and keys) and trimmed: the Android project with the edge-to-edge `MainActivity`, `store.js`, `dom.js` (screens, modal stack, toast), `fx.js` (synth sound, music, haptics), the icon sprite, the card/button kit and modal styles, the daily calendar, the level map layout (cities became flower beds), the deferred-save and win-recorded-at-once patterns, `tools/device.mjs`, `shot.mjs`, `release.mjs` (with the empty-file guard), `regress.mjs`'s harness and `make-art.sh`. Deleted: the word lists, dictionary, blocklist and word-search renderer.

New code is only what the arrow game needs: `logic.js` (generator, moves, solver, schedule), `garden.js` (12 flowers), the board renderer, tap input and arrow animations in `game.js`, `flowerSvg()` in `dom.js`, and new art in `sheet.html`.

## What exists
- **Rules (`logic.js`, pure, tested).** An arrow is `{ cells: [tail … head], d }`. It can leave iff its ray (the cells from the head to the board edge) is empty; the body follows the head along its own path, so only the ray matters. **Generation by reverse play:** arrows are added one at a time, each only where its ray is clear *at that moment*, so removing them in reverse order is always legal and every board is solvable by construction. The body grows by a random walk behind the head (first segment in line with the arrowhead, never into its own ray), lengths biased long; single-cell arrows are not generated (dull), so holes stay empty (~85% fill). **Any free arrow is always a safe move** (removal only frees space), so a player can never get stuck, only make mistakes; hints just pick a free arrow. Difficulty: board size and arrow length by level, and the best of 1–6 candidates by tangle depth (rounds needed when every free arrow is removed at once). Picture boards (heart, bloom, diamond; big heart, butterfly, mushroom, tree, tulip) every 5th level from 5.
- **Play (`game.js`).** Tap anywhere on an arrow's body (nearest body cell within 0.75 of a cell). It slides out along its track, clipped at the card edge, with a rising pentatonic note per exit. A blocked arrow slides up to the blocker, knocks (blocker flashes red, heavy haptic, wooden knock), slides back and costs a heart; 3 hearts per level, then "Out of hearts" → try again. Stars: 3 = no bumps and no hints, 2 = one, 1 = more. Hints glow a free arrow (one token, counts against stars; tapping hint again while the glow is up re-shows it free). Neighbouring arrows never share a colour (greedy colouring over 7 garden inks). Tutorial on level 1: the first free arrow glows for free.
- **Meta (`app.js`).** Home (a potted flower of the current bed), the garden (one bed per 10 levels, its flower once finished, tap it to re-read the card), the daily with a month calendar (a cleared daily gives a hint), settings, the Android back button. Hints start at 3; +2 per finished bed, +1 per first daily clear.

## Level stats (`npm test`, 1,000 levels)
| Levels | Board | Arrows | Tangle depth | Fill | Free at start |
|---|---|---|---|---|---|
| 1 | 4×4 | 4 | 2 | 88% | 2 |
| 2 | 5×5 | 9 | 4 | 84% | 6 |
| 3–4 | 6×6 | 11 | 2–5 | 86% | 5–8 |
| 5–12 | 7×6 | 10.4 | 3.4 | 85% | 5.6 |
| 13–25 | 8×7 | 12.3 | 4.2 | 87% | 5.3 |
| 26–50 | 9×8 | 14 | 4.4 | 87% | 6.9 |
| 51–100 | 10×9 | 15.4 | 4.7 | 86% | 6.6 |
| 101+ | 11×9 | 15.3 | 4.8 | 86% | 7 |

(Picture levels are included in the averages.) Generation: 2.4 ms average, 11–16 ms worst on desktop.

## Measurements (on the phone)
| Scenario | FPS | Late frames |
|---|---|---|
| Idle, 11×9 board | 60 | 0% |
| One arrow leaves (3 runs) | 59.2 | one 33 ms frame per tap (14 ms of sync work: sound + haptic + redraw) |
| Bump | 60 | 0% |
| Hint | 58.7 | one 50 ms frame |
| Petal confetti alone | 60 | 0% |
| Win flow | 57.1 | (test clears 17 arrows in one call; a player can't) |
| New level (generation + layout) | 29 ms of sync work, one 33 ms frame | |

Cell size on the phone: 72 px on level 1 (capped), 36 px on the 9-column boards.

## Bugs and fixes found by looking and testing
- **Level 1 had 9 arrows** (spec: about 5 seconds). The first generator drew short arrows and plugged holes with single cells. Now arrows are biased long and single cells are never placed: level 1 has 4.
- **Picture boards didn't read as pictures:** only empty cells showed a dot, so the tree was invisible under its arrows. Every board cell is now a pale tile, so the silhouette shows.
- **The hint glow's arrowhead was huge** (scaled with the glow width). Head size now has its own scale.
- **The bloom flower hid its own leaves.** The flower head is scaled down on the stem.
- **1.3× system text:** "Level 50 · Big heart" and then "Level 30 · Sunflower bed" truncated in the HUD; picture levels are now titled by the picture with "Level 30 · 10 of 10" below. The daily card's close button overlapped the month title; sheet headings now keep 30 px clear on each side.
- **White daisy on the cream icon card vanished at 48 px** (the §5.12 trap, again). The icon's bloom is now pink.

## Process problems (and the guard for each)
- **Empty stray files kept appearing** (`{`, `0`, `16`, `rows`, `cols)`, `setTimeout(r`, `!found[i]` …): 17 in this folder, 1 in the portfolio root and 2 in `wordpassport/` during this session. One (`www/16`) would have shipped; `release.mjs`'s guard refused the build and named it. Same cause as before (a Claude Code hook passes tool input through `cmd.exe`, which treats `>` as a redirect). Sweep with `find . -path ./node_modules -prune -o -path ./android -prune -o -type f -size 0 -print` before every release.
- **A bash heredoc containing JS failed** ("unexpected EOF while looking for matching `'`"). Patch scripts now go into a file (Write tool) and run with Node, as §5.10 says.
- **My first effect FPS numbers were confounded:** the script started a level inside the sampling window, so level generation looked like an exit-animation stall (one 399 ms frame). Start the level first, then sample only the effect.
- A desktop test tapped the same blocked arrow three times and "lost" only one heart. Not a bug: taps on an arrow that is still bumping are ignored. The test now uses three different arrows.

## Lessons for the next game
**What worked (keep doing it)**
- **Copying the last game whole, again.** The Android build succeeded on the first try; only the package id, names and art had to change outside the game code. Game 04 should copy this folder.
- **A generator that is correct by construction, then tested anyway.** Reverse play makes every board solvable; the test still solves 1,000 levels and 400 dailies and checks every path (connected, head in line, never in its own way).
- **Prove the rules are safe before designing hints:** "any free arrow is a safe move" meant hints, the tutorial and the fail message needed no solver at run time.
- **Printing boards as ASCII** (`peek.mjs` in the scratchpad) made generator tuning fast: fill, length and holes are visible at a glance without a browser.

**What to watch for**
- **The name check is the first task, not a formality.** The spec's name was taken four times over, plus two of my first alternatives.
- **Difficulty plateaus after level ~25** (about 15 arrows, depth ~4.7). The cell-size cap limits board size; more difficulty needs zoom, or more and shorter arrows on the same board. Tune it with real players.

## Commands
| Task | Command |
|---|---|
| Logic tests (content, rules, 1,000 levels, 400 dailies, timing, stats table) | `npm test` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start(40)"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Signed Play bundle (strips `__dbg` and `sheet.html` from the packaged copy) | `JAVA_HOME=… npm run release` → `android/app/build/outputs/bundle/release/app-release.aab` (3.5 MB) |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |

adb is not on the PATH: `$LOCALAPPDATA/Android/Sdk/platform-tools/adb.exe`.

## Not done yet (by decision)
- **Monetization:** ads, revive (a rewarded ad on the "Out of hearts" card), rewarded hints, skip level after 2 fails, interstitials from level 8. The heart and hint economy is the hook. Re-test layout with the real banner loaded.
- **Pinch-zoom and bigger boards** (the spec's 20×20). Only if players find the late game too easy.
- Spec variants: locked blocks, rotators, bombs. Add only if players ask or data shows a need.
- The picture album as a separate screen (the garden beds play that role for now).
- Store screenshots (`store/screens/`), hosting `store/privacy.html`.
- **Generator changes change levels.** Saved stars stay; an in-progress run whose arrow count no longer matches restarts fresh (guarded in `begin()`). Freeze `logic.js`'s generator once it ships.
- Unverified: music audibility, the launcher icon on a stock launcher, WebViews older than about Chrome 99 (`roundRect`).
