# Peak Lodge Solitaire (game 11)

Spec: `games/11-tripeaks-voyage.md` (TriPeaks solitaire). **A fresh project** (`npm init`, `npx cap init`, `npx cap add android`), every file written new (root DEVLOG §5.19). Ideas taken from earlier games: the insets bridge, the device tool and regression style, a board drawn on one canvas with flying pieces as small canvases (Lotus Mahjong, Tile Picnic), and static backgrounds on their own layer from the first day (Tile Picnic, §5.20).

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Peak Lodge Solitaire** | "TriPeaks Voyage" is effectively taken (**Solitaire Tripeaks - Voyagers**, **Solitaire Cruise**). Free (2026-10-21): Peak Lodge Solitaire, Three Summits Solitaire, Parlor Peaks. Chosen by the owner |
| Theme | A mountain lodge: every five deals climb a real summit | The spec's cruise with port postcards would repeat Word Passport's travel postcards (§5.16: games must differ). Suggested to and approved by the owner |
| Package id | `app.peaklodge.game` (**final**) | |
| Upload key | not made yet | Make `android/peaklodge-upload.jks` + `keystore.properties` before release |
| Style | **Vintage Analog** (from the owner's list): a 1950s lodge card room. Green baize, walnut rails, cream paper labels with a brass double rule like an old card box, Alfa Slab One for labels, Libre Baskerville for text and card indices; a teal card back with a cream lattice and a small mountain | The audience (40-75) and the lodge; unlike every earlier game |
| Board | One canvas for the three peaks (redrawn after a move), small canvases for the stock, the waste and the flying card | §5.20 |

## Outside resources used
| Resource | Licence | Use |
|---|---|---|
| **Byron Knoll's vector playing cards** (via notpeter/Vector-Playing-Cards): the 12 court cards | Public domain (README in `www/img/`) | The J, Q and K figures. The SVGs are 0.4-1.1 MB each (7.5 MB for 12), far too heavy for the P22, so they were rendered once in a desktop browser, cropped to the inner frame, toned a little warmer, and quantised to 96 colours: 248 KB for all 12 |
| **Wikidata**: names, heights and countries of 39 summits | CC0 | The journey and the journal. Queried 2026-10-21 by English name, restricted to mountains, the best-known match per name, Wikidata's preferred height where several are recorded (Everest 8,849 m, Mont Blanc 4,806 m, Vesuvius 1,281 m). Mount Etna left out: its height changes with eruptions |
| **Kenney** Casino Audio (card place, slide, shuffle; a chip for a wild) | CC0 | Card sounds |
| **Fontsource**: Alfa Slab One, Libre Baskerville | OFL | All text |
| Traditional and public-domain melodies (Ode to Joy, Twinkle Twinkle Little Star, Frère Jacques, Mary Had a Little Lamb) | public domain | The streak melody, as MIDI note numbers |

## Selling points (approved by the owner 2026-10-21, all four)
- **Big four-colour cards** (on by default, a setting): corner indices half the card's width; spades black, hearts red, diamonds blue, clubs green; suits drawn as paths, never font glyphs. Number cards show one large suit (the deck is for reading, not counting pips); court cards show the figure. At 360 px wide the cards are about 35 px across and still read at a glance.
- **A tune in every streak**: each card played sounds the next note of the level's tune on a small synthesized piano (a sine with two quieter overtones); from five in a row the note gets a fuller chord. Clearing the peaks plays the rest of the tune as a finale (at first it stopped after 16 notes, short of the end for the 26-note tune; now it plays to the end), and the win card names it.
- **A summit journal**: 39 summits from Scafell Pike (978 m) to Everest (8,849 m), lowest first, five deals each (195 levels); the fifth is the summit. The journal lists each summit with its height, country and the day reached, the total height climbed and how many Everests that makes, and the next summit; the home card draws the summits as a mountain range, reached ones filled in. Only data and arithmetic: no facts written from memory.
- **Household players**: up to eight named players on one device, each with their own progress, journal and game in progress; sound, vibration and the deck are shared. A name is outside input: control characters and markup characters removed, 1-20 characters, no duplicates.

## How it works
- **Rules** (`logic.js`, pure, tested): 28 places in four rows; a card is face up when the two cards on top of it are gone; it plays if it is one rank above or below the waste (K and A join up). Drawing ends the streak; every 8 in a streak earns a wild (anything plays on it). Three undos a deal.
- **Solver**: depth-first over (cleared places, cards left in the stock, waste top), remembering dead states, returning the winning moves. 189 of 200 random deals were clearable (it gave up on 9), about 23 ms a deal on the desktop.
- **Levels**: a deal is a seed (`www/levels.js` holds 195 numbers). A seed is kept if the solver clears it without wilds and a "decent" player (plays when it can, choosing the card that keeps a run going, else draws; uses wilds when stuck) wins it at a rate inside the level's band: easier on the low hills, harder towards Everest, the summit stage a little harder. Measured: levels 1-40 79%, 41-100 65%, 101-160 52%, 161-195 42%. 195 levels in 30-33 s.
- **The test replays the solver's moves for every level through the game's own rules** and checks they win.

## Measurements (on the phone, font scale 1.15)
| What | Result |
|---|---|
| Idle table (level 150) | 60.5 fps |
| 14 moves 300 ms apart (flying cards, melody notes), twice | 60.5 and 60.2 fps, no frame over 17 ms |
| 1.3× text | every screen fits after the fixes below |
| Device regression | 8/8: five runs clean, then two more after the last small fixes (`node tools/regress.mjs`): stage 1 with real taps following the solver's moves (cards and the stock), the win saved at once with one note per card; a misfit refused; a wild card; out of cards and undo; resume; the summit and its journal entry (dated today on the phone's calendar); a second player added by typing a real name on the keyboard, each keeping their own climb; the deck setting; back button |

## Bugs found by looking and testing
- **The journal could date a summit to the day before**: the day was stored with `toISOString()` (UTC), and before 05:00 on this phone that is yesterday. Found at 1.3× text in the small hours: the journal said 20 October on the 21st. Days are now stored and read on the phone's own calendar, and the regression checks the date.
- **My guessed Wikidata ids were mostly wrong** (one "Eiger" id was New York), which is why heights come from a query by name and type, not from memory.
- **The Met's API started answering with a bot check** after a burst of lookups while looking for antique playing cards. I stopped and dropped that idea rather than work around it.
- **Cards came out about 28 px wide** on a 360 px screen; tighter margins and a 1.45 ratio give about 35 px, and the big deck's corner index is now half the card's width.
- **"Deal again" wrapped** on its button, **the players list ran a name into its numbers** at 1.3× text, and it said **"1 summits"**. All fixed.
- **A loose test assertion** (the K-A wrap check would have passed with almost anything) was tightened to the exact cards.

## Commands
| Task | Command |
|---|---|
| Logic tests (rules, every level's solution replayed to a win, content) | `npm test` |
| Rebuild the level seeds (append-only once shipped) | `npm run levels` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start(40)"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |
| Signed bundle | `node tools/release.mjs` (needs `android/keystore.properties`) |

## Not done yet (by decision)
- **Monetization**: rewarded "+5 cards", wild cards for coins, interstitials. Wilds come only from streaks for now.
- **The spec's twists** (locked cards, bombs, colour sequences, two-sided cards), a daily deal, and removing a player.
- **The level file is append-only once shipped**: saves point at level numbers and move lists.
