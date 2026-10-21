# Clockwork Bolts (game 12)

Spec: `games/12-bolt-sort.md` (screw puzzle). A fresh project (root DEVLOG §5.19), every file written new. The owner chose the name, the Steampunk style and all four selling points (2026-10-21).

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Clockwork Bolts** | "Bolt Sort" is taken several times over (Bolt Sort Puzzle, Bolt Sort 3D, Bolt Sort - Stacking Puzzle). No app found for Clockwork Bolts, Tinker Bolts, Brass and Bolts or Bolt Workshop |
| Package id | `app.clockworkbolts.game` (final) | No upload key yet |
| Style | **Steampunk**: dark walnut backboard in an engraved brass frame, mahogany plates with brass grommets, enamel bolt heads, verdigris and oxblood; Cinzel Decorative, Cinzel, IM Fell English | Unlike every earlier game |
| Physics | **No rigid-body library.** A plate with one bolt left comes to rest with its centre straight below that bolt (exact), so each plate has only a few poses and "which pose covers which bolt" is a fixed table. The swing is an animation only | Deterministic on every phone, provable by the solver, and it removes the spec's biggest P22 risk (Planck.js/matter.js) |

## Outside resources
| Resource | Licence | Use |
|---|---|---|
| ambientCG Wood051 (backboard), Wood026 (plates) | CC0 | Textures (a "metal" texture was flat yellow; brass is drawn with gradients) |
| Kenney Impact Sounds, RPG Audio, Interface Sounds | CC0 | unscrew, blocked, creak, drop, tray, fall, bell, tick |
| Fontsource: Cinzel, Cinzel Decorative, IM Fell English | OFL | All text |

## Selling points (approved)
- **Fair physics + rewind**: the pose table above; every level is proven clearable without rewinds, and a level is rejected if any plate edge passes within 1.2 units of a bolt head's rim, so rounding can never decide a level. 3 rewinds a level; a rewound swing plays backwards.
- **Shape-coded bolts**: each colour has its own screw drive (slotted, cross, square, star, hex, triangle), on the bolt and on its box.
- **Build a working clockwork**: 10 machines x 12 parts (11 meshing gears and a finale). Music box (plays its tune on a synthesized tine), mantel clock, orrery, songbird; finished ones run in the workshop.
- **Slow-motion swing**: Calm swing slows swings and falls 2.6x; holding a bolt whose plate would move shows a ghost of where it goes (a slow tap on any other bolt still pulls, §5.20).

## Levels
120 levels built in 108 s (`npm run levels`), shipped as data (not seeds). Decent player's win rate: 1-30 97%, 31-60 77%, 61-90 67%, 91-120 55%; every 5th 62%. Eight bands could not be met in 4,000 seeds and were widened (levels 10, 15, 19, 23, 24, 26, 27, 99). Average 15.2 bolts and 6.9 plates, at most 10. `npm test` replays every level's solution through the rules.

## Measurements (P22, font scale 1.15)
| What | Result |
|---|---|
| Idle | 60.5 fps |
| Frame log, levels 70/100/115, 6 moves 2 s apart + 6 moves 0.4 s apart (~938 frames each) | 0, 1 and 2 frames of 33 ms; nothing longer |
| Device regression | 8/8, eight clean runs: real taps clear level 1 and the win saves at once; covered bolt; hold-to-preview (adb hold) + synthetic hold for the ghost; rewind by real tap; no-room card + rewind; resume; 12th part completes a machine and it runs in the workshop; calm setting; back button |
| 1.3x text | every screen fits |

## Bugs found and how
- The test found **two bolts of one plate too close** (the spacing check only looked at earlier plates).
- **A shipped level failed the test by 0.00006 units**: rounding to 5 decimals pushed a hanging corner past the board limit. The maker now checks one unit inside the limit the test uses.
- **The level maker could search forever** for an unreachable band: it now widens the band every 4,000 seeds and reports it.
- **33-67 ms frames during swings**: every frame drew about 50 small canvases, re-sent to the GPU each time. `ImageBitmap` copies made them 17 ms.
- **The win card had 249 ms frames**: it repainted 11 gradient gears every frame. Gear sprites made it 83 ms the first time and 33 ms after.
- **A "first move after a pause" hitch of 83-167 ms was mostly a measurement artifact**: the first frame of any sampling window started over DevTools is long. A rehearsal-flight "fix" looked like it worked, then didn't, and was deleted. Log frames with a 1 s idle lead-in and ignore the window's first frame.
- **Two changes made no measurable difference** and are recorded as such: the board canvas on its own layer, and clipped (dirty-rectangle) repainting, which a trace showed did not cut GPU raster (1,504 ms in 411 tasks against 1,200 ms in 262 before). The clipping is pixel-exact (checked against a full repaint after every move, `__dbg.same()`), but it is a candidate for deletion.
- A test edited the save and reloaded; the old page wrote its own copy back as it hid. Tests edit the live save object instead.

## Commands
| Task | Command |
|---|---|
| Logic tests | `npm test` |
| Rebuild levels (append-only once shipped) | `npm run levels` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 740 "__dbg.start(40)"` |
| Art | `bash store/make-art.sh` |
| Debug APK | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device tools | `node tools/device.mjs errors / eval / fps / shot`, `node tools/trace.mjs "<js>" [ms]`, `node tools/regress.mjs` |
| Signed bundle | `node tools/release.mjs` |

## Not done yet
- Monetization (end-game offer, boosters), the spec's Drill booster, a daily level.
