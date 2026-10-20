# Word Passport: Development Log

Game 02 of the portfolio (spec: `../games/02-word-trail.md`). A calm, travel-themed word search: 12 cities × 10 puzzles, a passport stamp and a postcard fact for each city, a daily puzzle, and bonus words that earn hints. Ads and other monetization are deliberately not built yet. Test device: Infinix X655C (Helio P22, Android 10, WebView 154, system font scale 1.15). Started 2026-10-20.


## v2 (2026-10-20): collage (a travel scrapbook), say hello, a local word, postcards
After the owner's feedback (root DEVLOG §5.16), restyled from scratch in a style from the owner's list and given three approved selling points.
- **Style: Collage.** Kraft paper, torn-edged paper scraps at slight angles, washi tape, stamps with perforated (dotted) edges, tickets with notched sides (a CSS mask), a taped polaroid of the next city on home with a handwritten hello. Courier Prime (typewriter) for text and the grid's letters, Caveat (handwriting) for titles and notes (both OFL). The grid card is a torn scrap with two strips of tape, baked once per layout.
- **A local word** (`local.js`): every puzzle also hides one word in the city's language (French SOLEIL, Japanese SAKURA, Aussie ARVO…), shown as "Also hidden: the French word for sun, 6 letters". It is placed like the listed words (so it reads in exactly one place), always last in `placed` so every index the game uses stays aligned, isn't needed to win, and fills two bonus-jar slots and is spoken aloud. 240 of 240 puzzles tested have one.
- **Say hello:** each city card has Hello and Thank you buttons in the local language (Bonjour / Merci, Konnichiwa / Arigatou with the Japanese script…), spoken by the phone's text-to-speech through `@capacitor-community/text-to-speech` (MIT; Android's WebView has no speechSynthesis). On the test phone 9 of the 10 languages have a voice; Afrikaans doesn't, and the game says so.
- **Postcards:** "Send a postcard" draws an airmail postcard (greeting, the city's fact, words found, the stamp and a postmark) on a canvas, writes it with `@capacitor/filesystem` and shares it with `@capacitor/share` (both official). The regression confirms the share sheet opens.
- **Outside resources:** the three plugins above; Kenney CC0 sounds (page flips for finds, a book landing as the stamp's thump, a click, a soft miss); Fontsource fonts.
- **Measured on the phone (v2):** idle 59.7 fps; four finds 400 ms apart 57.6 fps. Every screen fits at 1.3× text (after moving the hello note off the photo caption and fitting the half-width tickets). Regression 11/11, including a real swipe over the local word and the postcard's share sheet.
- **Bugs found (v2):** hiding the local word from the list would have shifted every word index after it (`found[i]` assumes `placed[i]` is `words[i]`); fixed by putting it last. The postcard's stamp didn't draw: an SVG used as an image needs `xmlns` (inline SVG doesn't) and an intrinsic size. A comment pasted mid-line by a patch commented out the code after it, twice in this restyle round.

## Day-one decisions (Gemstack retrospective §5.1)
| Decision | Value | Why |
|---|---|---|
| Name | **Word Passport: Word Search** | The spec's "Word Trail" collides with **Word Trails** (Netflix/PlaySimple) and **Word Trails: Search**. No "Word Passport" app found (2026-10-20). The passport fits the stamps loop |
| Package id | `app.wordpassport.game` (**final**) | Same per-game namespace pattern as Lotus |
| Upload key | `android/wordpassport-upload.jks` + `android/keystore.properties` (gitignored) | Own key per game, so each stays separable. **Back both up off this machine** and enrol in Play App Signing |
| Perf budget | 60 fps idle, ≥55 fps during effects, on the P22 | Measured below |
| Stack | Capacitor 8 + Canvas 2D (copied from Lotus) | Lotus proved it at 60 fps on this phone; a static letter grid is the easiest case there is |
| JDK | `JAVA_HOME` = JBR 21 at build time; nothing in `gradle.properties` | Portable |

## What was reused (the reuse-first rule)
Copied from `../lotus/` and trimmed, not rewritten: the Capacitor/Android project with the edge-to-edge `MainActivity` (inset fix + immersive mode), `store.js`, `dom.js` (screens, modal stack, toast), `fx.js` (synth sound, music, haptics), the icon-sprite pattern, the card/button kit and modal styles, the daily month calendar, the journey-map layout, the deferred-save and win-recorded-at-once patterns, `tools/device.mjs`, `regress.mjs`, `shot.mjs`, `release.mjs` and the `sheet.html` art pipeline (with Gemstack's ImageMagick mipmap recipe). Installed libraries: Nunito (already bundled), SCOWL via the `wordlist-english` npm package (dev dependency only; the app ships a generated text file).

New code is only what a word search needs: `logic.js` (grid generator + validation), `themes.js` (content), `blocklist.js`, the grid renderer and input in `game.js`, and the passport screens.

## What exists
- **Rules (`logic.js`, pure, tested).** Seeded generator: picks theme words that never contain one another (forwards or backwards, so PAN inside PANCAKE can't be ambiguous), places longest first with crossings about half the time, fills with English letter frequencies, then **validates**: every listed word readable in exactly one place in all 8 directions, and no blocked word readable anywhere except inside a listed word (BASS may contain a blocked substring). Difficulty ramp: 6×6 with 4 words across/down → 10×8 with 11 words in all 8 directions by puzzle 200. Columns capped at 8 so letters stay big.
- **Content (`themes.js`).** 113 general themes and 12 city lists (2,290 words), one true fact per city. The first world tour never repeats a theme; later tours reshuffle themes and stay at the hardest shape.
- **Play (`game.js`).** Swipe in any of 8 directions (snapped), **or tap the first letter and then the last** (for players who find swiping hard). Highlighter capsules, a rising pencil-tick while swiping, a pentatonic run on finds. Bonus words (any real 4+ letter word, from a 29,933-word list of common words, SCOWL levels 10–35) fill a jar; 5 = a free hint. Each bonus word counts once per puzzle, even across restarts (`save.bonusLog`), so it can't be farmed. Hints ring a word's first letter; when every word left is ringed, a hint flashes a whole word. Stars: 3 = no hints. A 2-step tutorial on puzzle 1.
- **Meta (`app.js`).** Home (passport cover), the passport (one page per city, stamps, puzzle nodes, tap a stamp to re-read its postcard), the daily puzzle with a month calendar (a solved daily also gives a hint), settings, the Android back button. City finish: stamp "thump", postcard fact, +2 hints.

## Measurements (on the phone)
| Scenario | FPS | Late frames |
|---|---|---|
| Idle, 10×8 grid | 59.9 | 0% |
| Real finger swipe across the grid (adb input) | 60 | 0% |
| Word found (steady state) | 58.4–59.2 | one 33 ms frame per find, with or without sound |
| Bonus word (+ flying chip) | 58.1 | 3% |
| Hint | 58.7 | 1% |
| Confetti alone | 60 | 0% |
| Win flow / city-stamp flow | 57.8 / 57.1 | (test finds all words in one frame; a player can't) |
| New puzzle (generation + layout) | one 72–83 ms frame at the screen change | |

Generation: 3.6 ms average, 60 ms worst on desktop for 1,000 puzzles (it was 27/210 ms before `validate` stopped rebuilding every line string for each word). Cell size on the phone: 54 px at puzzle 1, 40 px on the largest grid (39 px at 1.3× system text).

## Bugs found by testing on the device
- **A find animation outlived the puzzle.** Quitting within 300 ms of a find crashed the effect, because cell geometry read `G` after it was cleared. Geometry now keeps its own column count (root cause fixed in `center()`, the one function every effect uses); pointer handlers also guard a mid-swipe quit. Found by the regression's page-error check.
- **1.3× system text:** the word list grew so tall it shrank the grid to 32 px cells; tightened chips (now 39 px). "Continue puzzle 11" and "Postcard from Mexico City" wrapped or truncated; the label is now "Continue" with the puzzle in the subtitle, and the city puzzle is titled with the city.
- The postcard stamp's curved city name collided with its outer ring, and long countries ("United Kingdom") overflowed the inner ring. The country now runs along the bottom arc, like a real postmark.
- The cover stamp on the home screen was navy ink on a navy booklet. Removed: stamps live on the passport pages.

## Process problems (and the guard added for each)
- **Empty stray files** named after code fragments (`{let`, `!g)`, `[gx` …) kept appearing in the project, including `www/`, so they would have shipped in the APK. Their timestamps match edits whose text contained `=>`: a Claude Code hook in this environment passes tool input through `cmd.exe`, which treats `>` as a redirect. Lotus's root has the same junk. `tools/release.mjs` now refuses to build if `www/` contains an empty file (Capacitor's own empty `cordova*.js` stubs excepted). The hook itself is outside this repo.
- **CRLF line endings** crept in from Python patch scripts (text mode on Windows), and the release script's `__dbg` stripper (LF-only pattern) then failed. Its safety check caught it; files were normalised to LF and the pattern now accepts `\r?\n`. Patches are now made with the Edit tool or Node, not Python.
- A test waited 1.9 s for a card that opens at 2.0 s. Timings in `regress.mjs` now cite where they come from.

## Lessons for the next game
**What worked (keep doing it)**
- **Copying the last game whole.** Android setup, saves, sound and device tools from Lotus worked on the first build. Nearly all new work was word-search-specific: the generator and the content. Game 03 should start by copying this folder, not Lotus.
- **Checking the name before writing anything.** "Word Trail" was taken. Found on day one, so nothing had to be redone.
- **Validating content automatically.** The spell-check of every theme word against SCOWL flagged 8 of 2,290 words (3 real words SCOWL spells differently, 5 replaced). Any game with written content (words, facts, levels) should get a test like this.
- **The on-phone regression with a page-error check.** It caught the quit-during-animation crash, which manual play would almost never hit.

**What to watch for**
- **Time anything that runs when a screen opens, not just animations.** Puzzle generation was 27 ms on average and 210 ms at worst on desktop: a visible pause on the P22. One fix made it about 8× faster. Measure generation and layout as well as fps.
- **Large system text breaks layouts that look fine at normal size.** Check 1.15× and 1.3× on the phone before calling any screen done.
- **Art that reads at full size can fail when tiny or on a matching colour.** The icon's postmark hid a letter at 48 px, and a navy stamp disappeared on the navy cover. Preview icons at launcher size, and check contrast against what's behind.
- **Design the reward loop before ads exist.** Hints cost stars, bonus words earn hints, and a restart can't farm them. Rewarded ads can then plug into the hint button without changing the game's rules.

**What I'd do differently**
- **Make the theme order stable from day one** (for example, sort themes by a fixed hash of their name), so adding themes after launch never changes existing puzzles. Today it does (see "Not done yet").
- **Get real players early.** Difficulty, the hint economy and the word lists are tuned by judgement only. Fun can't be measured here, so a few closed-test players are worth more than further polish.

## Commands
| Task | Command |
|---|---|
| Logic tests (content spell-check, 1,000 puzzles, 400 dailies) | `npm test` |
| Rebuild the bonus dictionary | `npm run dict` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 760 "__dbg.start(150)"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Signed Play bundle (strips `__dbg` and `sheet.html` from the packaged copy) | `JAVA_HOME=… npm run release` → `android/app/build/outputs/bundle/release/app-release.aab` (4.0 MB) |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |

adb is not on the PATH: `$LOCALAPPDATA/Android/Sdk/platform-tools/adb.exe`. If `adb devices` is empty, re-plug or re-authorise USB debugging (it was empty for the first ~30 minutes of this session, then appeared).

## Not done yet (by decision)
- **Monetization:** ads, rewarded hints, "Remove ads". The hint economy (start 5; +1 per 5 bonus words, per daily, +2 per city) is the hook rewarded ads will plug into. Re-test layout with the real banner loaded (Gemstack §5.10).
- Spanish/Portuguese editions (the spec's biggest growth lever: separate word lists, not translations).
- Weekly themed events, more cities, a stats page. Add only if players ask or data shows a need.
- Store screenshots (`store/screens/`), hosting `store/privacy.html`.
- **Content updates change puzzles.** Adding themes reshuffles the tour order, so existing puzzle numbers get new grids. Saved stars stay; an in-progress run whose word count no longer matches restarts fresh (guarded in `begin()`). Append cities freely; think before adding themes after launch.
- Unverified: music audibility, the launcher icon on a stock launcher (the test phone uses a text-only launcher), behaviour on WebViews older than about Chrome 99 (`roundRect`).
