# Last Bus Home: Color Sort (game 09)

Spec: `games/09-bus-rush-jam.md`. **A fresh project**: `npm init`, `npx cap init`, `npx cap add android`, and every file written new. On the owner's instruction (2026-10-20: "copy from different game but only take inspiration, not copy paste the whole everything"), no earlier game folder was copied. Earlier games were read for ideas only: passing the system-bar insets to the page (Shopkeeper Trio's `MainActivity`), a device tool over adb and DevTools, an on-device regression with real taps, the save-the-win-at-once rule.

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Last Bus Home: Color Sort** | The spec's "Bus Rush Jam" is taken in all but word order (**Bus Jam Rush**, **Bus Rush: Traffic Jam**, **Bus Rush! : Traffic Jam**); **Bus Stop Shuffle** is a console game. Free (2026-10-20): Last Bus Home, Commuter Colors, Night Bus Sort, Mind the Queue, Route Riders. Chosen by the owner |
| Package id | `app.lastbushome.game` (**final**) | |
| Upload key | not made yet | Make `android/lastbushome-upload.jks` + `keystore.properties` before release |
| Style | **Flat Design**, as transit wayfinding (from the owner's list): navy signage bars with white and amber type, flat route colours, a metro map of 45-degree lines and round stop dots, pictogram passengers, a yellow tactile strip at the platform edge. Overpass (after the US highway signage face) for all text. No shadows, no gradients | A commuter game reads as signage; unlike every earlier game |
| Colour and letter | Every route has a letter (A-F) on passengers and buses | Colour is never the only cue |
| Board | DOM: tiles a static CSS grid; one element per passenger, moved by `transform` along their real walking path | |
| Levels | Pre-built in `www/levels.js` by `tools/make-levels.mjs` | |

## Outside resources used
| Resource | Licence | Use |
|---|---|---|
| **dariusk/corpora**: `humans/firstNames` (US Census 2000s; the upstream file has a missing comma, so names are pulled out by pattern) and `geography/london_underground_stations` (265 unique names; two names are two stations each) | CC0 (stated in its README) | Passenger names and stop names. Place names only: no roundel, no network branding |
| **Kenney** Impact Sounds (concrete footsteps), RPG Audio (bus doors), Interface Sounds (click, no) | CC0 | All sounds except a two-note synth station chime |
| **Fontsource**: Overpass 400/700/800 | OFL | All text |
| **Capacitor** app, haptics (official) | MIT | Back button, haptics. Nothing is shared, so no share or filesystem plugin |

## Selling points (approved by the owner 2026-10-20, all four)
- **Your transit map.** 12 lines of 20 stops, drawn as a metro map: routes in 45-degree steps from fixed seeds, so the map is the same every time. A line appears when you reach it; every stop you clear becomes a station dot you can tap to ride again; the next stop pulses with its name in a sign.
- **Passenger stories.** Every passenger has a name and a reason to travel, fixed per level and spot (a hash of level and index). When a bus leaves, the ticker names its three riders and tells one story ("Route B left with Kiara, Christian and Giselle. Kiara is on the way to fix a neighbour's bicycle."); the win card shows three. The 60 trip reasons are written never to guess anyone's pronouns, and a test checks that.
- **Night service.** From 19:00 to 06:00 on the phone's clock (or always day / always night in Settings), the station turns dark, bus windows are lit amber, and sounds play at half volume. Rechecked every minute.
- **Call one back.** Once per stop, send someone on the bench back to their spot. The stuck card offers it while it's unused.

## How it works
- **The key simplification**: boarding is automatic, so the street (which bus is at the stop, how many seats are taken, who is on the bench) follows from **which** passengers have left, not the order. The whole game state is a set. `street()` computes it from that set; the screen uses an explicit event version (`tap()`: walk to the bus or bench, depart, arrive, board from the bench), and the test checks the two agree after every tap of random games, call-backs included.
- **Solver**: depth-first over taps, remembering dead sets; passengers who board at once are tried first. Worst level: 58 ms (95 ms in another run) on the desktop.
- **Curation** (`tools/make-levels.mjs`): each slot asks for its station size, colours, buses, pillars, bench seats and hidden passengers, and a band for how often a careless player (any allowed tap, at random, 40 tries) clears it. Every 5th stop from 10 on is hard: one bench seat fewer and a lower band. Measured careless win rate: stops 1-3 100%, 4-20 69%, 21-60 46%, 61-140 27%, 141-240 11%. 240 levels in 29 s, one process.
- **Walking**: the path is a breadth-first search through empty cells to the front row, then up to the bench seat or the bus door, as one WAAPI keyframe animation on `transform`.
- **The queue**: walks to the bench start at once; anything to do with the bus (walking aboard, leaving, the next arriving, boarding from the bench) runs in order on a promise chain, so quick taps never wait for animations.

## Measurements (on the phone, font scale 1.15)
| What | Result |
|---|---|
| Idle station (level 200) | 60.4 fps |
| 12 taps 300 ms apart, with buses leaving and arriving | 59.7 fps (worst frame 50 ms) |
| Home, the map with the pulsing stop | 60.2 fps |
| 1.3× text | every screen fits after the fixes below |
| Device regression | 8/8, five runs in a row clean (`node tools/regress.mjs`): stop 1 with real adb taps, the win card's stories and the next bus at the stop, the win saved at once and drawn on the map, a blocked tap and the bench, call one back with real taps, a stuck card from random play, resume, night service (by setting and by the clock), back button |

## Bugs found by looking and testing
- **Queued bus animations ran against the next stop** (intermittent page errors in 3 of 8 regression runs): a restart or a new stop left the previous stop's chain running. First fix: every queued step belongs to the game that queued it. That left a second path: a walk already under way finished after the new stop began and looked up its element in the new list; now a walk hides its own element. Five runs in a row clean after that. Found by running the regression repeatedly.
- **After a cleared stop, the next stop had no bus**: the last bus's drive-away kept its final frame (`fill: 'forwards'`) and held the new bus off screen. Found in a store screenshot; `drawBus` cancels the bus's old animations, and the regression now checks the bus is at the stop after "Next stop".
- **The win card never appeared**: the win is recorded the moment the last passenger leaves (`G.over`, root §5.11), and the queued step that shows the card returned early on `G.over`.
- **The story of the last stop stayed on screen at the next one.**
- **At 1.3× text** the win title was cut off, and the "Night service" badge squeezed the title, then wrapped, then overflowed the departure board. Card titles now wrap, and the badge moves under its label when there is no room.
- **The upstream names file is not valid JSON** (a missing comma) and **the station list repeats two names**; both handled when the content is generated, and the test checks the stop names are unique.
- **Store art came out as a zoomed corner**: the screenshot tool captured a 1024-pixel canvas through a 512-pixel viewport. `shot.mjs` now saves a canvas's own PNG when the page returns a data URL.
- **Empty stray files** again (the `cmd` redirect hook, root §5.10): 7. Deleted.

## Commands
| Task | Command |
|---|---|
| Logic tests (all 240 levels solvable and replayed through the event rules, random play agrees with the set rules, content) | `npm test` |
| Rebuild the levels (append-only once shipped) | `npm run levels` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start(45)"` |
| Icons, splash, store art | `bash store/make-art.sh` |
| Debug APK onto the phone | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device: errors / eval / FPS / screenshot | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]`, `shot out.png` |
| Device regression (after every change) | `node tools/regress.mjs` |
| Signed bundle | `node tools/release.mjs` (needs `android/keystore.properties`) |

## Not done yet (by decision)
- **Monetization**: rewarded extra bench seats, boosters (helicopter, shuffle, extra bus), interstitials. Call one back is free once per stop for now.
- **VIPs, tunnels, frozen groups** from the spec, city skins, and a daily stop.
- **A hint.** Not built: the solver is fast enough to run on the phone, but call one back and restart already cover mistakes. Easy to add.
- **The level file is append-only once shipped**: saves point at stop numbers.
