# Quiet Sudoku: Classic & Killer (game 07)

Spec: `games/07-sudoku-calm.md`. Engine copied from `wordparlour/` (Android project, saves, sound and haptics, modal stack, device tools); the look, the rules and every screen were written from scratch.

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Quiet Sudoku: Classic & Killer** | The spec's "Sudoku Calm" is taken (an app of that name), and so are **Calm Sudoku** and **Zen Sudoku**. Free (2026-10-20): Quiet Sudoku, Sudoku Teahouse. Chosen by the owner |
| Package id | `app.quietsudoku.game` (**final**) | |
| Upload key | not made yet | Make `android/quietsudoku-upload.jks` + `keystore.properties` before release |
| Style | **E-Ink UI** (from the owner's list): an e-reader page. Warm grey e-paper, black ink, four greys, no colour, no shadows, no gradients; hairline and 2px rules, square corners, a reader's status bar, a contents list with dotted leaders; a press inverts (paper on black); changing page flashes black like an e-paper refresh; a night page inverts the palette. Literata (Google's e-book serif) for text and the player's numbers, Atkinson Hyperlegible (made for low vision) for printed numbers | The audience (35–75) reads; an e-reader is calm and legible, and unlike every earlier game |
| Board | DOM: 81 cells built once, patched per change; killer cages are one static SVG | §5.15: a patched DOM board is fast on the P22 |
| Puzzles | Pre-built into `www/pack.js` by `tools/make-pack.mjs` (2,000 puzzles, 305 KB) | Making a hard puzzle takes many tries; the phone never generates |

## Outside resources: used, and tried and rejected
| Resource | Licence | Use |
|---|---|---|
| **jsPDF** 4.2.1 (`www/vendor/jspdf.umd.min.js`, 420 KB, loaded only when a book is made) | MIT | The puzzle book |
| **The Metropolitan Museum of Art, Open Access**: 12 woodblock prints (10 Hiroshige, 2 Hokusai; object ids in `www/art.js`) | Public domain, CC0 | The monthly postcards. Dithered to four greys (Floyd–Steinberg, ImageMagick) so they look like an e-ink screen; 297 KB for all 12 |
| **Kenney** Interface Sounds (click, scratch, error, confirmation, tick) and RPG Audio (book flip) | CC0 | Placing, pencil notes, a mistake, a finished row/column/box, page turns |
| **Fontsource**: Literata (variable), Atkinson Hyperlegible 700 | OFL | All text and numbers |
| **@capacitor/filesystem** 8.1.5, **@capacitor/share** 8 | MIT (official) | Write the PDF to the cache, hand it to the share sheet |
| `sudoku-core` 3.0.3 (MIT): **rejected** | | One "hard" puzzle took 13.3 s to generate on the desktop; its grader returned no difficulty for 95 of 200 valid puzzles (beyond its techniques), and so no hint for them; a hint took 113 ms on the desktop; its steps don't say which cells they rely on, which teach-me hints need |
| `killer-sudoku-generator` 1.1.2 (MIT): **rejected** | | Fast (22–39 ms), but its killers keep 31–51 printed digits and use 6–7-cell cages; killer players expect few or no digits and 2–4-cell cages |
| Art Institute of Chicago images: **not used** | | Their API data is CC0, but the image server answers scripts with a Cloudflare bot check; I didn't try to get around it and used the Met's API instead |

## Selling points (approved by the owner 2026-10-20, all four)
- **Teach-me hints.** The solver (`sudoku.js`) only uses techniques a person uses, easiest first: only place in a box/row/column, only number left, cage sums, pointing, locked in a box, pairs, hidden pairs, triples, X-wing, Y-wing. Every step carries the cells it relies on and one plain sentence. A hint is the solver's steps from the board the player sees (mistakes left out) up to the next number: the area is hatched, the reasons are boxed, the answer square is outlined, and "Write the 6" places it. Consecutive cage steps fold into one "What the cages allow" step with one worked example, and a rare long chain becomes "Clearing the way" plus the last three steps, so a hint is at most 4 steps (tested). Eliminations walked through also leave the player's notes.
- **Print a book.** Six puzzles of a chosen kind from the far end of the pack (not the next ones the app deals), two to an A4 page in large print, killer cages and sums drawn, answers on page 4; `book.js`. Saved to the app cache and shared.
- **A monthly postcard.** This month's print is covered by 24 pieces; each solved puzzle (any kind) uncovers one, in an order fixed per month (`card.js`). Finished months stay in an album.
- **Gentle mode** (off by default): no mistake limit, no clock, a wrong number turns grey instead of being struck through.

## How it works
- **Generator** (`makeClassic`): fill a grid by randomised backtracking, remove clues in symmetric pairs while the answer stays unique (a solution counter, most-constrained cell first). 5–13 ms per puzzle on the desktop.
- **Grading**: solve with the technique solver; the hardest technique sets the band. Easy: singles only, 34+ clues. Medium: singles only, under 32 clues. Hard: pointing, locked, pairs. Expert: triples, X-wing, Y-wing. Puzzles the solver can't finish are thrown away, so **every shipped puzzle has one answer, is solvable by logic, and every hint can explain**. In 1,000 puzzles made with at least 22 clues: singles only 66.4%, level 2 6%, level 3 2.1%, level 4 3.9%, unfinished 21.6% (before Y-wing was added, Expert was rarer).
- **Killer** (`makeKiller`): cut a solved grid into 2–4-cell cages with no repeated digit, then add printed digits only where the solver gets stuck, until it finishes. 2–13 digits (median 5 of 40); the full killer keeps at most 6, the gentle one gets 8 more. The solution ships in the pack (solving a killer at start took up to 146 ms on the P22).
- **Soundness test**: replaying every step of 240 puzzles, no technique ever removes the true digit and every placement is the true digit; every hint from a half-filled board ends in the right number.
- **Rules** (`play.js`, pure): right numbers clear that number from the notes of the row, column and box; undo restores them too; three mistakes offer one more chance or a restart; the clock runs only while the board is visible and no card is open. A saved game resumes only if its printed digits still match the pack.
- **Daily**: days since 2026-01-01 around a 400-puzzle daily pack of medium and hard puzzles, the same for everyone.
- **Text sizing**: the board's font size comes from the cell size divided by the WebView's text zoom (measured with a 100px probe), so digits and notes always fit their squares at any system text size; "Extra-large numbers" makes them bigger.

## Measurements (on the phone, font scale 1.15)
| What | Result |
|---|---|
| Idle board | 60 fps |
| 8 numbers 300 ms apart | 59.5 fps (worst frame 50 ms) |
| The win (ink wipe) | 58.9 fps; 35.5 ms of one-off work at the moment of solving |
| Starting a game | easy 3.5 ms (worst 10.4), expert 11.5 (worst 37.3), killer worst 4.9 ms (146 ms before the solutions shipped) |
| A hint | expert 0.1 ms; killer from a fresh board 65.7 ms |
| Making a book | 113 ms |
| Pack build (desktop) | 138 s; `npm test` checks all 2,000 puzzles in about 8 s |
| 1.3× text | every screen fits after the fixes below |
| Device regression | 11/11 (`node tools/regress.mjs`): real taps for squares and numbers, notes and undo, teach-me, the mistake limit, the win saved at once with its postcard piece, resume, killer cages and hints, gentle mode, back button, the book's share sheet |

## Bugs found by looking and testing
- **Killer hints ran 26 steps before placing a number** (one cage at a time). Folding cage steps and summarising long chains brought every hint to at most 4 steps (median 1).
- **The merged cage step boxed every cell on the board.** Merged and summary steps now mark only their example.
- **The win sweep animated 81 cells' background** (main-thread paint, 51.8 fps) and **forced a layout once per cell**. Now one ink layer moves by transform (58.9 fps), and unit sweeps restart with one forced layout.
- **Killer start hitch**: solving at start took up to 146 ms on the P22; solutions now ship in the pack (puzzles unchanged; the tool checked).
- **At 1.3× text** the statistics table ran off the right edge (fixed columns, "Avg.") and the merged cage hint was cut off (at most four sets spelled out, filler sentence removed). The regression now checks the killer hint's text fits its card.
- **A class name clash**: the week row's `today` dot took the daily card's `.today` styles (padding, border). Renamed `now`. (§5.15's "scope selectors" again.)
- **The page-refresh flash fired on the first reveal**, and a screenshot caught the page black. It now flashes only on a page change.
- **Cage lines ran through the sums.** The first cell of each cage leaves room for its sum.
- **The store screenshots showed the phone's status bar with notification icons** (`adb screencap` captures the whole screen). Cropped to 720×1424 (system bars off, within Play's 2:1 limit).
- **Registering the SystemBars plugin twice** printed a warning; Capacitor already registers it, so the app uses `Capacitor.Plugins.SystemBars`.
- **ImageMagick's `PNG8:` prefix stopped Git Bash converting `/d/...`** to a Windows path; use `D:/...` after a format prefix.
- **10 empty stray files** again (the `cmd` redirect hook, §5.10). Deleted.

## Commands
| Task | Command |
|---|---|
| Logic tests (all 2,000 puzzles, soundness of every technique, hint length, play rules) | `npm test` |
| Rebuild the puzzle pack (append-only once shipped) | `node tools/make-pack.mjs` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start('hard')"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |

## Not done yet (by decision)
- **Monetization**: rewarded second chance and hints, interstitials between puzzles, "Remove ads". Hints and the second chance are free for now.
- **An "Extreme" band** (puzzles needing chains or swordfish): the solver would need those techniques first, so their hints could explain them.
- **Digit-first input** (pick a number, then tap squares) and long-press notes.
- **Killer techniques beyond cage combinations** (the 45 rule, innies and outies): killers get extra printed digits instead, so the solver can always finish and explain.
- **The pack is append-only once shipped**: saved games and the book counters point at indexes.
