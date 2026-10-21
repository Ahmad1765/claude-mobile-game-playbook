# Ludo Courtyard (game 13)

Spec: `games/13-ludo-royale.md` (Ludo). A fresh project (root DEVLOG §5.19), every file written new. The owner chose the name, the Skeuomorphism style and all four selling points (2026-10-21), then asked mid-build for **co-op** (chosen: teams of two) and **online play** (on a new Supabase account the owner connected).

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Ludo Courtyard** | "Ludo Royale" is taken (Ludo Royale - Board Game by MakHub, LUDO ROYALE 3D by omek, an iOS app). Free: Ludo Courtyard, Ludo Chowk, Ludo Lanterns, Ludo Bazaar |
| Package id | `app.ludocourtyard.game` (final) | No upload key yet |
| Style | **Skeuomorphism**: an embroidered cotton chaupar cloth on a dark wooden table; dyed patches multiplied over a real weave, stitched dashed seams, gold stars, a braided edge; lacquered wooden tokens; bone dice with red one and four. Rozha One for display, Baloo 2 for everything else | Familiar to the audience (South Asia, Middle East, Africa); both fonts also cover Devanagari for later localization |
| Dice | `crypto.getRandomValues` with rejection sampling (bytes below 252), the same for people and computer players | The genre's top complaint is rigged dice; this is the trust selling point |

## Outside resources
| Resource | Licence | Use |
|---|---|---|
| ambientCG Fabric062 (cloth), Wood066 (table) | CC0 | The board cloth and the table |
| Kenney Casino Audio (die throw, chips) and Interface Sounds | CC0 | roll, step, capture, home, win, no |
| Fontsource: Rozha One, Baloo 2 | OFL | All text |
| @supabase/supabase-js 2.118 (its single-file browser build, 218 KB) | MIT | Online rooms (Realtime broadcast + presence) |

## Selling points (approved)
- **Fair-dice ledger**: every roll is counted per player (people by name, the computer by level) the moment it is made, saved at once, and shown as bars against the fair line with the share of sixes. Online, guests count the host's rolls too.
- **Named house rules**: three sixes lose the turn, safe stars, capture before home, quick start, teams of two; saved under a name (e.g. "Nani's rules"), one tap in the seats screen.
- **Move coach**: a red glow under your tokens in danger; the coach's pick has a green ring and a reason from the score part that decided it ("it captures a token", "it reaches a safe square"). The bots use the same scoring.
- **Match story**: a race chart (each player's steps after every turn, captures marked), the first capture, the most captures, the comeback, lost three-sixes turns; shared as a 1080×1350 picture.
- **Added on request, co-op**: teams of two (partners opposite, no capturing your partner, a finished player moves the partner's tokens, the team wins together).
- **Added on request, online**: rooms with a six-letter code. The host's phone runs the rules and rolls every die; guests only send "roll" and "pick", which the host checks against whose turn it is and what is legal. Empty seats can be filled by the computer; a player who leaves is replaced by the computer; a player idle for 30 s has the move made for them. Online games aren't saved for resuming.

## How it works
- **Rules** (`rules.js`, pure): the 52-square track from a run-length description; places relative to each seat (−1 base, 0-50 track, 51-55 home column, 56 home, and 100 + square for laps under "capture before home"). Six to come out, exact roll home, captures and arriving home earn a roll, three sixes lose the turn.
- **Bots** (`bot.js`): easy is random; normal is the score with noise; hard weighs danger (opponents 1-6 squares behind, turning off into their own column excluded).
- **Online** (`online.js`): one public Supabase Realtime channel per room. Every message is outside input: a state from the host is rebuilt field by field (player kinds and seats, every token place, die, turn, rules as booleans, story bounded), names are cleaned, anything else is dropped. ponytail: public channels mean anyone with a code can join or send; checks make that harmless to the game, and private channels with anonymous sign-in are the upgrade path. The Supabase security advisor reports no issues (the project has no tables).

## Measurements
| What | Result |
|---|---|
| `npm test` | rules; 1,000 bot games all end with only legal moves, **Hard beat Easy in 91.8%**, Normal beat Easy in 81.8% of 400; 300 team games all end with both winners home |
| P22, four computer players at normal speed, 20 s | 60 fps, no frame over 20 ms; with a board full of tokens, one 33 ms frame |
| Device regression (`node tools/regress.mjs`) | 8/8 in five runs in a row, then 9/9 (with teams) in four more: a new game by real taps (Start, the die) and the roll in the ledger; a real tap picks the token; capture and extra roll; the coach; named rules typed on the real keyboard (quick start); resume with the die kept; win and story; ledger and back; teams of two |
| Online, desktop host + desktop guest (`node tools/online-test.mjs 30`) | the guest joined by a lower-case code; 30 rolls, both boards identical, the guest's ledger counted all 30; garbage, a forged state and forged requests from an outsider changed nothing; a guest who left was replaced by the computer |
| Online, the phone hosting + a desktop guest (`node tools/online-device.mjs 25`) | joined in 2.4-3.7 s; a teams game with two computer players. **Before the fix below, 4 of 15 runs ended with the guest one roll behind** (every send had reported ok; one more run lost the DevTools connection). **After it, 4 of 4 agreed** (the test now also waits up to 12 s for the boards to agree). When the host left, the guest returned home |
| 1.3× text | every screen fits |
| A full game, four computer players, Classic rules, quick bots (on the phone) | 443 rolls, 26 captures; its real story and ledger are in the store screenshots |

## Bugs found and how
- **A guest could miss a message and stay one roll behind** (4 of 15 phone runs). The host's sends all reported ok, so the likeliest cause is a lost broadcast (Realtime broadcast does not promise delivery); not proven. The host now re-sends the whole state every 3 s, and a step that fails on a guest no longer blocks the messages after it (a rejected promise chain would have dropped them all silently). 4 of 4 runs agreed after.
- **The game froze for good** after a move: a frame's timestamp can come before the `performance.now()` read when the hop starts, the time went negative, the path index became −1 and the error killed the animation loop. Found by a 40 s automatic game in the desktop preview (stack traces added to the preview tool). Clamped at zero.
- **The ledger cut its own heading** to "All the dice tog": headings went through the 16-character name cleaner. Headings are now escaped, not shortened. Found by the regression.
- **"1 rolls"** (ledger and win line): pluralised.
- **The die was invisible** until the first roll; a blank face is drawn now.
- **My first `target()` for "capture before home" was garbage**, caught on rereading before any test ran; laps got their own encoding.
- Three test mistakes, each caught by running them: four sixes where three were meant, the 3 × 3 home's geometry, and test hooks that didn't save as the real code paths do.
- **I copied the previous game's preview tool** at first, against the owner's no-copy rule; it was rewritten for this game.

## Commands
| Task | Command |
|---|---|
| Rules, bots, teams | `npm test` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "<js>"` |
| Art | `bash store/make-art.sh` |
| Debug APK | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device | `node tools/device.mjs errors / eval / frames / shot`, `node tools/regress.mjs` |
| Online | `node tools/online-test.mjs [rolls]` (desktop), `node tools/online-device.mjs [rolls]` (phone hosts) |
| Signed bundle | `node tools/release.mjs` (also removes the `__dbg` and `__net` hooks, so a release can only roll fair dice) |

## Not done yet (by decision)
- Monetization (interstitials, themes for coins), the daily spin, localization (Hindi, Urdu, Bengali, Indonesian, Arabic, Portuguese).
- Online: private channels with sign-in, public matchmaking with strangers, chat, rejoining a game after a dropped connection.
- The Supabase project `ludo-courtyard` (Mumbai) is on the owner's new account; the free plan's limits (concurrent connections, messages a second) bound how many games can run at once.
