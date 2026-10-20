# Word Parlour: Development Log

Game 06 of the portfolio (spec: `../games/06-word-groups-solitaire.md`). A word-association solitaire in the style of Solitaire Associations Journey: category cards open slots, word cards go to their category or stack on each other in the columns, a draw pile, limited moves. **Design style: Art Deco.** Three selling points, approved by the owner: "why it fits" notes, make a deal for a friend, a share-grid daily. Ads and other monetization are deliberately not built yet. Test device: Infinix X655C (Helio P22, Android 10, system font scale 1.15). Started and reached a device-tested v1 on 2026-10-20 (regression 10/10).

## The owner's feedback mid-game, and what changed
While this game was being built, the owner said the games were "way too similar to each other in ui", "the ui looks way too ai generated", that I wasn't using online resources (GitHub, asset packs) to go faster, and that every game needs a selling point that sets it apart. Then: choose a named design style per game. What changed:
- **Copy the engine, not the look.** This game's `index.html` was rewritten from scratch in one style (Art Deco) instead of patching the shared kit.
- **Outside resources first**: real recorded sounds, open fonts, the official Share plugin (list below).
- **Selling points, approved before building** (below). The style chosen for every game is in the root DEVLOG §5.16.

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Word Parlour: Word Solitaire** | "Word Groups" is taken (**Word Groups: Match & Sort**, **Word Groups: Brain Puzzle**); "solitaire" names are crowded (Word Solitaire Now!, Solitaire Word Association…); "Kindred Words" taken. Free (2026-10-20): Word Parlour, Teatime Words, Word Tableau. Chosen by the owner |
| Package id | `app.wordparlour.game` (**final**) | |
| Upload key | not made yet | Make `android/wordparlour-upload.jks` + `keystore.properties` before release |
| Style | **Art Deco**: midnight navy, gold leaf, ivory cards with a double gold rule, jade category cards, oxblood badges; square plaques with double rules, a faint sunburst; Josefin Sans (geometric, period-correct) for text, Poiret One for the title only; no soft shadows, no gradient buttons | The audience (40–70) and the "parlour" name; deliberately unlike every earlier game |
| Board | DOM (cards are text) | Mosaic Parade and Shopkeeper Trio showed a DOM board patched per change is fast on the P22 |

## Outside resources used
| Resource | Licence | Used for |
|---|---|---|
| Kenney "Casino Audio" (kenney.nl) | CC0 (licence file checked: `www/sfx/KENNEY-LICENSE.txt`) | Card place, slide, shuffle, fan and chip sounds, replacing the synth beeps |
| Josefin Sans, Poiret One (via Fontsource) | SIL OFL (`www/fonts/OFL-*.txt`) | All text |
| `@capacitor/share` (official plugin) | MIT | The native share sheet |
| `wordlist-english` (SCOWL, via npm, dev only) | SCOWL licence | Spell-checking the category bank in the test (Word Passport's approach) |
| Considered for later: dariusk/corpora (CC0 word lists by category) | CC0 | More categories |

## Selling points (approved 2026-10-20)
1. **Why it fits**: every finished group shows a one-line note (`www/notes.js`, 82 notes, well-established facts only); the win card lists them all.
2. **Make a deal for a friend**: an editor (3–4 groups, 3–6 words each, the draft kept in the save); "Send to a friend" shares a code (`WP1.` + base64url) through the share sheet; "Play a friend's deal" takes a pasted code. Decoding validates everything (it's outside input). A friend's deal earns no stars and doesn't touch progress.
3. **Share-grid daily**: after the daily, one coloured square per group in the order found (green quick, yellow steady, orange slow, against the level's average moves per group), stars and moves to spare. The native share sheet opened on the phone.

## How it works
- **Rules** (`logic.js`, pure, tested). K category slots, columns (top card face up), a stock and a waste. The moving unit is a column's top run of face-up cards of one category (a category card only at its bottom) or the waste's top card. An empty slot takes a run starting with its category card; a started slot takes that category's words; a column top takes words of its category; an empty column takes anything. Every action, draws included, costs a move; a complete category flies away and frees its slot.
- **Content rules** (`categories.js`, enforced by the test): every word in exactly one category of the bank, 2–8 letters, spelled as in SCOWL (a short hand-checked list of proper nouns and loanwords aside: Gouda, fusilli, piñata…); words that genuinely fit two categories left out (KIWI is a fruit and a bird); 38 pairs of categories that overlap by nature (Gems/Colors, Halloween/Fantasy…) never dealt together. **Needs a human read-through before release** (spec 06).
- **The greedy player** (`pick`): words to their category, category cards to free slots, moves that turn a hidden card over, waste cards onto matching columns, else draw; it never shuffles between columns for nothing. Solver, tutorial glow, test bot. Deals it can't finish are thrown away; the move limit = its moves × a slack that tightens with level (1.8 → 1.15) + 2.
- **Input**: tap to move. A card with one place to go (or its category's slot) goes there; with several, it is selected and its places light up. Tap the deck to draw.
- **Helpers**: one token for an undo (any time) or 5 extra moves (when out); 3 to start, +2 per room, +1 per first daily. "No moves left" offers an undo.

## Balancing, measured (`npm test`, 500 levels; careless player on levels 1–200)
| Levels | Groups | Cards | Greedy moves | Move limit | Careless player wins |
|---|---|---|---|---|---|
| 1 | 2 | 8 | 10 | 20 | 100% |
| 2–5 | 3 | 14 | 19 | 32 | 100% |
| 6–10 | 4 | 20 | 28 | 39 | 100% |
| 11–20 | 4 | 24 | 29 | 37 | 75% |
| 21–40 | 5 | 30 | 42 | 53 | 38% |
| 41+ | 6 | 36 | 54 | 65 | 20% |

Careless player: half greedy, half random legal moves, inside the move limit. The bots know every card's category and people don't, so people will find it harder than this. 82 categories, 595 words. Generation: 1.5 ms average, 9 ms worst on desktop; 17–24 ms on the P22.

## Measurements (on the phone, font scale 1.15)
| Scenario | Result |
|---|---|
| Idle, level 45 | 60 fps, 0% late |
| 15 actions 150 ms apart, before the restyle | 58.3 fps, 2.7% late, worst 50 ms (typically one 33 ms frame per action) |
| Same, after the Art Deco restyle | **60 fps, 0% late, worst 16.8 ms** |
| One action's work (median), with or without sound and haptics | 10–11 ms; render 8–12 ms of it |

## Bugs found by looking and testing
- **Words shrunk to fit grew back at 1.3× text.** `fit()` wrote a pixel size; Android's text zoom scales pixel sizes again (15.8px became 20.5px). Writing the size in `em`, relative to the card, applies the ratio exactly. Found by measuring the labels on the phone after a first fix didn't work.
- **The boot waited for a font that no longer existed** (Nunito, removed in the restyle), so the page showed before Josefin loaded and `fit()` measured the fallback font. Boot waits for the real fonts, and the table re-renders on `document.fonts.ready`.
- **The tutorial glow vanished on the phone**: the `ResizeObserver` re-renders the table after the first layout and the glow classes went with it. `render()` re-applies the glow. Desktop timing hid it.
- **The tutorial said "tap the purple category card" when none was showing**: the tip now follows the greedy player's next move (draw / open a slot / place a word / stack).
- **The room card would have crashed from the rooms screen** (a blanket rename made it read the current game, which is null there). Caught on review.
- **The win card showed "Back to the parlour" twice** for the daily.
- **The icon's thin Poiret W vanished at 48 px**: the icon uses bold Josefin (the §5.12/5.13 trap again).
- **A `sed` put a comment mid-line**, commenting out the text alignment after it. Caught in the preview.
- **Bash heredocs failed on apostrophes** twice ("Time's", "plant's"); content files go through the Write tool.
- **A share-sheet screenshot showed the owner's contacts.** Deleted at once; the listing notes: never take store screenshots with a share sheet open.
- **Empty stray files**: 15 here and 4 at the portfolio root. Deleted.

## Lessons for the next game
- **Pick the style first and write the page from scratch in it**; copy only the engine. Name the style in the DEVLOG.
- **Search for CC0/MIT resources before drawing, synthesizing or writing content.** Real card sounds took minutes.
- **Size text in `em` when JS shrinks it**: px is scaled again by the WebView's text zoom.
- **Wait for the fonts you actually use**, and re-measure when they arrive.
- **Content needs written rules and a test**: unique words, no genuine doubles, clashing categories never together, spell-checked.

## Commands
| Task | Command |
|---|---|
| Logic tests (content and notes rules, spelling, table rules, friend codes, 500 levels, 400 dailies, 100 friend deals, careless-player difficulty) | `npm test` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start(45)"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |

## Not done yet (by decision)
- **Monetization**: ads, rewarded extra moves and undos, interstitials.
- Clever wordplay categories (Connections-style "___ ball") for late levels, with notes; more categories from dariusk/corpora.
- A "report this word" button (spec risk table), store screenshots, the upload key, hosting the privacy page.
- **Generator changes change levels**: saved runs resume only when the level's signature matches. Freeze `categories.js` order once shipped (append-only).
