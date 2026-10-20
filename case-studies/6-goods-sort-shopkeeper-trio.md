# Shopkeeper Trio: Development Log

Game 05 of the portfolio (spec: `../games/05-shelf-sort.md`). A goods triple-sort in the style of Goods Sort: shelves of three slots, a hidden layer behind each; move items between shelves, three of a kind on one shelf clear, an empty shelf brings its next layer forward; clear everything before the clock runs out. Every 10 levels a new aisle (product set), a daily, an "extra time" booster. Ads and other monetization are deliberately not built yet. Test device: Infinix X655C (Helio P22, Android 10, system font scale 1.15). Started and reached a device-tested v1 on 2026-10-20 (regression 9/9).

## v2 (2026-10-20): neo-brutalism, customer orders, your shopfront
After the owner's feedback (root DEVLOG §5.16: the games looked alike and "AI generated", no outside resources, no selling point), this game was restyled and given two approved selling points.
- **Style: neo-brutalism.** `index.html` rewritten from scratch: flat lemon/tomato/mint/sky/pink blocks, 3 px black outlines, hard 4 px offset shadows that press flat when tapped, a dot-grid paper backdrop, Archivo Black for display and Space Grotesk for text (OFL), a sticker-style logo. The goods' outlines became heavier and pure black to match. The home is now your shopfront with chunky colour blocks, not a hero over stacked buttons.
- **Customer orders** (selling point 1). From level 3 (and in the daily) a shopper walks in (a Kenney CC0 animal) and asks for one item; only items with 3+ copies left can be ordered, so every order is servable. Popping a trio of it while they wait (30 s, on the game clock) adds 8 s to the clock and pays a 5–10 coin tip (quicker = bigger). The wanted item is tinted on the shelves.
- **Your shopfront** (selling point 2, `shopfront.js`). An SVG facade on the home screen: every finished aisle puts two of its goods in the window, and tips buy 8 decorations (sign lights, flower boxes, bunting, striped awning, bench, lamp post, shop cat, neon OPEN sign) on the Decorate screen.
- **Outside resources:** Kenney "Animal Pack" (customers), "Interface Sounds" and "RPG Audio" (drops, chimes, coins, a door), all CC0 with their licence files in `www/img/customers/` and `www/sfx/`; Archivo Black and Space Grotesk via Fontsource (licences in `www/fonts/`). Credits in Settings.
- **Measured on the phone (v2):** idle 60 fps; a burst of 12 moves with a shopper present 56 fps (4% late); per event, moves 17–33 ms median frame, pops 33 ms median (67 worst), a serve 67 ms. Regression 11/11 (customer served with a tip; decorations bought, refused without coins).
- **Bugs found (v2):** the customer row appearing and leaving resized the stage and shifted every shelf (rule §5.11 again): its space is now reserved for the whole level. A `#customer img` rule meant for the animal also stretched the ordered item to 64 px in a 40 px box (found by measuring the boxes on the phone). `overflow-wrap: anywhere` broke "SHOPKEEPER" and "DECORATE" mid-word at 1.3× text: smaller type instead. The shop sign's SVG text named the font "Archivo Black" while the page's `@font-face` is "Archivo", so it fell back to plain sans.

## Day-one decisions (Gemstack retrospective §5.1)
| Decision | Value | Why |
|---|---|---|
| Name | **Shopkeeper Trio: Goods Sort** | The spec's "Shelf Sort" is taken many times over: **Shelf Sort: Cozy Sort Puzzle**, **Shelf Sort Match: Goods Puzzle**, **Shelf Sort Master: Goods Match**, **Shelf Sort: Goods Sort Puzzle**, **Shed Shelf Sort**. Also taken or too close: Tidy Shelves, Pantry Sort, Market Sort, Shelf Shuffle, Bazaar (two Bazaar triple-match games), Trinket Shelf (*Trinket Tidy*). Free on Play (2026-10-20): Grocer Trio, Little Grocer Match, Shopkeeper Trio. Chosen by the owner |
| Package id | `app.shopkeepertrio.game` (**final**) | Same per-game pattern |
| Upload key | not made yet | Make `android/shopkeepertrio-upload.jks` + `keystore.properties` before the first release |
| Perf budget | 60 fps idle, ≥55 fps during effects, on the P22 | Met: see Measurements |
| Stack | Capacitor 8; the board is **DOM**, not canvas | The board is static except for the dragged item and pops; DOM gives layout, hit areas and accessibility for free (Mosaic Parade showed DOM tokens moved by transform cost nothing on the P22) |

## What was reused (the reuse-first rule)
Copied `../mosaicparade/` whole (minus build outputs, keys and node_modules): the Android project, `store.js`, `dom.js`, `fx.js` (synth sound, music, haptics), the icon sprite, the card/button/modal kit, the daily calendar, the level-map layout (album pages became shop aisles), the deferred-save and win-recorded-at-once patterns, per-mode resume, the booster-on-the-fail-card flow, `tools/*`, `make-art.sh`. The Android debug build passed on the first try for the fourth game running.

New code: `items.js` (36 SVG goods in 6 aisles), `logic.js` (shelves, stuck rule and tidy, greedy player, generator, schedule), the shelf renderer, drag and tap input, pops and combos and the clock in `game.js`, the shop screen in `app.js`, new art in `sheet.html` (with `?art=items`, a contact sheet of every item at real size).

## How it works
- **Shelves** (`logic.js`, pure, tested). A shelf is a front row of 3 slots plus layers behind it. Move a front item to any free front slot; three of a kind in a front row pop; an empty front row takes the next layer. Every type has a multiple of 3 copies, so a win empties everything.
- **Never stuck, proved first** (rule §5.13). With one free slot any arrangement of the front rows is reachable (the free slot works as a buffer), so a type with 3+ copies up front can always be popped. Otherwise a shelf must be emptied to bring a layer forward, which needs 3 free slots. So `stuck()` = no type with 3 up front *and* fewer than 3 free slots (with a layer still behind). When stuck, the game tidies a shelf by itself, for free: it swaps 3 copies of the most common type (from any layer) onto one shelf and pops them. That always works because every type left has a multiple of 3. Exactly-full shelves aren't the only deadlock: 1–2 free slots with no triple up front is one too.
- **The greedy player** (`pick`) scores every move: finishing a triple, building a pair on a clean shelf, emptying a shelf with a layer behind, moving the odd one out of a shelf that holds a pair whose third copy is up front; never undoing the last move. It is the solver, the clock's measure, the tutorial's glow and the regression's bot. The solver treats a repeated front-row state as stuck and tidies, so it always ends.
- **Levels.** K item types from the level's aisle (earlier aisles fill in when K > 6), 3 copies each (6 for a few), shelves 4–9, 3–4 free front slots, the rest in layers behind (some 2-item layers). No front row may start as a triple. **The clock** = 4 s per move the greedy player needs (5 s up to level 10) + 20 s, rounded up to 10 s.
- **Input.** Drag (the item is its own element moved by `transform`; drop anywhere on a shelf picks its free slot nearest the finger), or tap the item and then the spot (the easier input, §5.12).
- **Rendering.** The shelves are built once per level; after that only slots whose item changed are touched. Items are 96 px PNGs rasterised from the SVGs once at boot. Pops are three copies puffing up on an overlay layer; a new layer slides forward. Combos (pops within 4 s) climb the scale and show "Combo ×n".
- **Meta** (`app.js`). Home shows three goods of the current aisle on a shelf. The shop: one aisle per 10 levels, its six goods lighting up as its levels are cleared. Extra time starts at 3; +2 per finished aisle, +1 per first daily. Stars: half the clock left or more = 3, a fifth = 2, less or extra time used = 1. The clock starts at the first move and stops when paused or in the background.

## Balancing, measured (`npm test`, 500 levels; the careless player on levels 1–200)
| Levels | Items | Greedy moves | Greedy tidies | Clock | Careless player wins |
|---|---|---|---|---|---|
| 1 | 9 | 7 | 0 | 60 s | 100% |
| 2–5 | 17 | 11 | 0 | 78 s | 88% |
| 6–10 | 24 | 16 | 0 | 100 s | 50% |
| 11–20 | 33 | 28 | 0.6 | 135 s | 40% |
| 21–40 | 42 | 36 | 1.2 | 167 s | 40% |
| 41+ | 48 | 39 | 0.93 | 180 s | 25% |

"Careless player": half its moves greedy, half random, one move per 2.5 s, wins if it clears the shelves inside the clock. **A purely random player wins 0%, even on level 1** (random moves never build triples), so it measures nothing in this genre; the half-greedy one does (the §5.14 rule needed adapting). Generation: 3.0 ms average, 12 ms worst on desktop; 18–29 ms on the P22.

## Measurements (on the phone, font scale 1.15)
| Scenario | Result |
|---|---|
| Level generation + first render (levels 12, 25, 48, 77, 207) | 18, 29, 19, 19, 20 ms |
| Idle, level 45 (9 shelves) | 60 fps, 0% late |
| A real finger drag (`adb shell input swipe`) | 60 fps, 0% late |
| 12 moves 150 ms apart (several pops), first version | **46.3 fps, 10.4% late, worst 100 ms** |
| Same, after updating only changed slots | 53.3 fps, 3.3% late, worst 183 ms |
| Same, after rasterising items to PNG once | **58.7 fps, 2.3% late, worst 33 ms** |
| One pop, per move: SVG images vs PNG | worst frame 67–100 ms vs 17–50 ms |

Every screen checked at 1.15 and 1.3 text: nothing truncated; the aisle card's title and button wrapped at 1.3 and were shortened.

## Bugs found by looking and testing
- **Items were drawn about 30 px wide** with three shelves per row on a 360 px phone. Two shelves per row: about 52 px, and nine shelves still fit a 640 px-tall screen.
- **The back layer overlapped the front items' tops.** It is now a small preview strip above the front row.
- **The greedy player looped** (level 9 hit the 400-move cap): a shelf held `0,1,0` with the third 0 elsewhere, and nothing rewarded moving the stray 1 out. Added that score, and the solver treats a repeated state as stuck.
- **Pops cost 67–100 ms frames.** Each move rebuilt every shelf's HTML, and every new `<img>` decoded and rasterised its SVG again (and again at the puff's larger scale). Updating only changed slots and drawing from PNGs rasterised once at boot brought a burst of moves from 46.3 to 58.7 fps. Found by measuring the worst frame per move type (plain move / pop / pop with a new layer) instead of per burst.
- **The store art's donuts were tiny**: the donut fills only 65% of its icon box, so boxes a third of the shelf wide drew donuts a fifth wide. (Also gave the SVGs an intrinsic 40×40 size, or canvas `drawImage` uses a 300×150 default box.)
- **Test mistakes, not game bugs**: a hand-made test fixture had 6 items, not 7; a star-boundary test expected 2 stars for 10 s of 60 (the rule says 1). Fixed the tests after checking the rule.
- **A bash heredoc failed on an apostrophe** ("Time's up") inside the patch script, as on Arrow Garden; patch scripts go through the Write tool now.
- **Empty stray files**: 9 at the project root and 1 at the portfolio root, none in `www/`. Deleted.

## Lessons for the next game
**What worked (keep doing it)**
- **Copying the last game whole, a fourth time.** Only the game code was new.
- **Proving the never-stuck rule before writing the rescue.** Working out exactly when a position is dead (not just "no free slots") made the tidy a few lines, and the test exercises it.
- **Measuring the worst frame per kind of event**, not per burst: it pointed straight at pops.

**What to watch for**
- **A purely random player is useless as a difficulty measure in sort games**: use a half-greedy one.
- **Many SVG `<img>` elements are expensive on the P22** when they are created often or scaled; rasterise once to PNG.
- **Item art is the retention risk the spec names.** 36 items is enough for 6 aisles; more aisles need more art (append-only).

## Commands
| Task | Command |
|---|---|
| Logic tests (content, rules, stuck/tidy, 500 levels, 400 dailies, careless-player difficulty, timing, stats table) | `npm test` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start(45)"` |
| Item contact sheet at real size | `node tools/shot.mjs "sheet.html?art=items" items.png 720 760` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Signed Play bundle | `JAVA_HOME=… npm run release` (needs the upload key first) |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |

adb is not on the PATH: `$LOCALAPPDATA/Android/Sdk/platform-tools/adb.exe`.

## Still to do
- Play it by hand: clock length, whether 2 layers deep is enough challenge late on, whether the tidy feels like help or like cheating.

## Not done yet (by decision)
- **Monetization:** ads, rewarded "+60 s" on the time-up card, rewarded boosters, interstitials. The extra-time economy is the hook.
- The spec's other boosters (magnet, shuffle, freeze), coins, the shop-renovation meta, locked/frozen/mystery items, a no-timer relax mode. Add if play-testing shows a need.
- Store screenshots, hosting `store/privacy.html`, the upload key.
- **Generator changes change levels.** A saved run resumes only if the level's signature matches (`sig()` in `game.js`); otherwise it starts fresh. Freeze the generator and `items.js` order once it ships (append-only).
