# Gobble Town (game 14)

Spec: `games/14-swallow-city.md` (hole / swallow arcade). A fresh project (root DEVLOG §5.19), every file written new. The owner chose the name, the Memphis style and all four selling points (2026-10-21).

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Gobble Town** | "Swallow City" and the alternatives turned up no app; the owner picked Gobble Town |
| Package id | `app.gobbletown.game` (final) | No upload key yet |
| Style | **Memphis**: cream ground with a dot grid, lilac roads, mint parks; buildings in flat bright colours with chunky black outlines, hard offset shadows and roof patterns; toy-like street things; confetti backdrop; Bungee and Outfit | A bold toy city that reads top-down at small sizes |
| Engine | Pure `logic.js` (world grid, holes, swallow rule, growth `r = sqrt(R0² + K·eaten)`, bots, holes eating holes), fixed 60 Hz steps | Tests drive it without a screen |

## Outside resources
| Resource | Licence | Use |
|---|---|---|
| **OpenStreetMap** via the Overpass API: buildings, roads, parks, water, trees, benches, lamps, bus stops, signals, fountains, bins, post boxes, phone boxes, bike racks | ODbL 1.0, attribution "© OpenStreetMap contributors" (shown on the home card, in every round, in Settings, the listing and the feature graphic) | The towns. Fetched once, 2026-10-21, one request per city with pauses (`tools/fetch-osm.mjs`, cached in `data/osm`); one 502 stopped the run, and the rest were fetched once more a few minutes later |
| Kenney Interface and Impact sounds | CC0 | pops, gulp, grow, eaten, end, tick |
| Fontsource: Bungee, Outfit | OFL | All text |

## The towns
Seven 460 × 660 m neighbourhoods: Le Marais (Paris), Shibuya (Tokyo), Greenwich Village (New York), Sultanahmet (Istanbul), Downtown (Cairo), Fort (Mumbai), Lagos Island (Lagos). **Lahore was tried twice and dropped**: OSM has 7 buildings in the Walled City and 1 in Model Town. `tools/make-cities.mjs` turns the OSM data into `www/cities.js` (858 KB): simplified building outlines (kind from size and tags; named places of worship, historic and tourist buildings are landmarks), roads by width, parks, water, street things. The game adds parked cars, vans, scooters and buses along drivable roads, and street clutter, all from a seed. Built in a few seconds.

Balance came from tests, not guesses:
- **The size ladder.** A swallow game needs things at every size. Cairo's and Mumbai's holes stopped at 8 and 13 m wide until mid-size things were added: kiosks, market stalls and vans (2.4-3.6 m), garages (4-6 m) and pavilions (6.5-10 m), placed only where there is room.
- **The opening.** With 350 small things spread anywhere, a new 4 m hole met one every ~30 s in Tokyo. Now 1,200 small things per town (street clutter along the pavements), and a round starts at the busiest bit of pavement near the middle. Test: a hole steering for the nearest food swallows 13-30 things in its first 15 s in every town.
- **The finale.** Every object must be swallowable in calm mode: the largest object's radius must be under 92% of the radius a hole reaches by eating the whole town. Tokyo failed (a 111 m complex against a final 119 m); buildings over 60 m radius are left out (4 in Tokyo, 2 in Mumbai, 1 each in New York and Istanbul).
- Six bots for two minutes: the biggest reaches 20-66 m radius depending on the town; Cairo and Mumbai are the hard ones (4-7% of the town eaten), Istanbul the easiest (46%).

## Selling points (approved)
- **Real city maps**, with landmark names popping up as they are swallowed.
- **Two holes, one phone**: split screen, each half its own camera; the top half is turned upside down for the player opposite; each finger steers its own hole; the pause button sits on the dividing line.
- **Sticker book**: 26 kinds, with counts, and a list of landmarks by town.
- **Calm mode**: no clock, no rivals, the whole town; progress (which things are gone, the hole) is saved every 2 s and resumes after a relaunch.

## Drawing, and the P22
Tiles of 256 px at five zoom levels (1.5-24 px a metre), in two layers: the ground, and everything standing on it. The holes are drawn between them, so something too big to swallow covers a hole's edge with no extra drawing; each hole's rim is drawn again, dashed, on top, so a hole under a building stays visible. A swallow clears and repaints only its own small area, at the zoom level in use. At most one new tile is drawn per frame, and the farthest level is drawn as a round starts (behind "Drawing Paris…": 161 ms on the P22 for Paris).

How it got there (each step measured on the phone, frames logged after a 1 s idle lead-in):
1. First version (one tile layer at 2x density; every swallow redrew whole tiles; the hole's edge objects redrawn by hand every frame): a small hole 58.5 fps, a big one **13.7 fps** (frames up to 765 ms).
2. Two layers, local repaint, at most two new tiles a frame: a small hole fell to 40.5 fps. A trace showed the GPU saturated.
3. **The game canvas at 1x** (360 × 800 instead of 720 × 1600): a small hole **59.7 fps**. Measured on a fresh app's first round, so not affected by the bug below.
4. Later steps were measured wrongly at first: **every round, and every resume, started another animation loop** that drew whatever round was current, so each round played made the next slower (nine rounds, nine loops). Only the first round after a launch was clean. After the fix (one loop per round, tied to its id), all sizes were steady across rounds.
5. With the loop bug fixed: ImageBitmap tiles made no difference (59.9/58.4/56 fps against 60/58.3/56.4) and were **removed**. Releasing tiles between rounds (`dispose`) is kept.

Final (P22, font scale 1.15), each case in two towns:

| Case | Paris | New York |
|---|---|---|
| Small hole (battle, 8 s) | 59.1 fps | 59.7 fps |
| A 54 m hole | 58.4 fps | 60 fps |
| A 150 m hole | 57.4 fps | 57.9 fps |

Two players on one phone (two viewports, Tokyo): **54.8 fps**, just under the 55 target.

## Bugs found and how
- **Half the buildings were drawn black**: the colour hash returned a signed number (`^=`), and a negative palette index gave no colour, so the black shadow showed. Found at 1.3× text in a two-player round in Mumbai; fixed with `>>> 0`.
- **A hole under a building it can't swallow vanished**: found in the first two-player screenshot; the dashed rim on top fixed it.
- **The loop-per-round bug** above, found because measurements got worse with every round played.
- **The calm save made as the app hides did not always reach storage** before Android stopped the app: the regression lost a few seconds of progress. Calm now saves every 2 s.
- **"19 M"** on the results: Bungee has only capitals; numbers use Outfit. Milestones say "metres wide" and use the width, as the HUD does.
- **The results clock showed 0:01**: the HUD now refreshes when a round ends.
- **Two-player HUDs overlapped**, and player 2's upside-down HUD sat under the status bar: fixed (top inset, pause on the divider, each player's clock in their own pill).
- Test mistakes, each caught by running it: a straight-line opening test (a hole driven straight through building blocks finds nothing; replaced by the bots' own steering), and a calm test that started one town and resumed another.

## Commands
| Task | Command |
|---|---|
| Download the OSM extracts (once; cached) | `node tools/fetch-osm.mjs [id ...]` |
| Build the towns | `npm run cities` |
| Rules, balance, openings | `npm test` |
| Desktop screenshot | `node tools/shot.mjs out.png "<js>"` |
| Art | `bash store/make-art.sh` |
| Debug APK | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device | `node tools/device.mjs errors / eval / frames / shot`, `node tools/trace.mjs "<js>" [ms]`, `node tools/regress.mjs` |
| Signed bundle | `node tools/release.mjs` |

The device regression (`node tools/regress.mjs`) passed 6/6 on five runs in a row, then once more on the final build:
- a real drag steers the hole and it swallows things;
- battle results, with the stickers saved at once;
- the sticker book;
- calm mode resumes after a relaunch;
- two players: each half steers its own hole, and the top half's direction is turned round;
- pause and resume, and the back button.

## Not done yet (by decision)
- Monetization (rewarded skins, a bigger start, interstitials), hole skins, daily quests, the spec's puzzle mode.
- Moving traffic and people (left out on purpose: no people to swallow).
- Lahore, until OSM has its buildings.
