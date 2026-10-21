# Bun Street (game 15)

Spec: `games/15-burger-rush-idle.md` (idle arcade, "carry stacks" restaurant). A fresh project (root DEVLOG §5.19), every file written new. The owner chose the name, the Retro-Futurism style and all four selling points (2026-10-21). The last of the 15 games.

## Day-one decisions
| Decision | Value | Why |
|---|---|---|
| Name | **Bun Street** | No app is called exactly "Burger Rush Idle", but "Burger Rush" is in use (a mode in IDF: Burger & Fries, an older game); "Idle Diner" is taken. No app found for Bun Street, Burger Lane or Patty Parade |
| Package id | `app.bunstreet.game` (final) | No upload key yet |
| Style | **Retro-Futurism**: a 1950s googie diner on a night street. Checkered floor (the paint colour at a third, over cream), chrome counters, starbursts and a boomerang table, cherry red and mustard, the joint's name in neon on a dark panel. Righteous and DM Sans | Unlike any other game in the portfolio |
| Economy first | `logic.js` (pure) and a simulated good player in `test/economy.test.mjs`, before any visuals, as the spec asks | The curve is tuned from measured times to each pad |
| Drawing | The room drawn once per city/paint into a background canvas; people, food, pads and cash drawn each frame; the canvas at CSS pixels | Root DEVLOG §5.24 |

## Outside resources
| Resource | Licence | Use |
|---|---|---|
| Kenney Interface Sounds, RPG Audio (coins) | CC0 | pick, drop, cash, buy, serve, pay, city |
| Fontsource: Righteous, DM Sans | OFL | All text |

## Selling points (approved)
- **Build-to-order**: each customer's ticket (the city's main filling plus up to two open extras) shows in a bubble as a little burger; the counter builds it once it holds those fillings.
- **Street-food tour**: Mumbai (vada pav), Lahore (bun kebab), Lagos (suya burger), Tokyo (teriyaki burger), each with its own fillings and a higher price. One plain line describes each dish, nothing more.
- **Name and paint your joint**: a cleaned name (1-18 characters), four paint jobs, the neon sign in the room, and a shareable postcard.
- **Fair idle**: offline earnings only with every open station staffed and a cashier, at half of the cash collected per minute over your last two minutes of play, for up to two hours; no ads, no purchases at all.

## The economy (measured by the simulated player)
| City | Pads | Time | Longest wait for a pad |
|---|---|---|---|
| Mumbai | 15 | 12.3 min | 88 s |
| Lahore | 17 | 16.3 min | 118 s |
| Lagos | 17 | 16.4 min | 112 s |
| Tokyo | 16 | 14.2 min | 106 s |

Pads cost ~1.17x the one before. The first pads take 20-50 s each, as the spec asks (20-60 s); the last few take 60-118 s, longer than the spec's range but steady. The next city's pad appears only once everything else in a city is bought.

The simulated player found two design faults before any screen existed:
- **Hands full for good.** A player holding fillings the counter had no room for (its tray full) was stuck: they could not put them down or pick anything else up. Now there is a bin.
- **The wrong station.** Walking past the raita station on the way to the chutney filled your hands with raita the counter could not take, and the player looped between the raita station and the bin for good. A station now only hands you what the counter has room for, counting what you already carry.

## Measurements (P22, font scale 1.15)
| What | Result |
|---|---|
| A shift with the room at 2x | **44-46 fps**, steady 33 ms frames |
| The same at 1x (§5.24) | **60 fps**, no long frames; 59.5 fps on the final build with the autopilot buying a pad every few seconds |
| Device regression (`node tools/regress.mjs`) | 7/7, five runs in a row, then once more on the final build: real drags walk the chef (pick up, put down, a customer served, cash collected, the cashier pad bought); resume after a relaunch; the name typed on the real keyboard and a paint chosen; fair idle (an hour away at half of $120 a minute: $3,600 plus the relaunch's seconds); the tour moves on to Lahore; back button |
| 1.3× text | every screen fits after the fixes below |

## Bugs found and how
- The two economy faults above (simulation).
- **Two stations both read "chutney"** and two helper pads "Helper: chutney": labels used the last word of a filling's name. Full names now, on two lines when needed.
- **The neon sign did not glow** on the cream wall: it hangs on a dark panel now.
- **People and food were too small**, and the full-strength checker drowned them: bigger sprites, a softer floor.
- **The HUD pill was cut off** at 1.3× ("Lahore · Bun keba"): it shows the dish only.
- A paused kitchen was redrawn 60 times a second under the card: it is drawn once.
- Test-tool mistakes, each found by checking the evidence rather than retrying:
  - **An adb swipe with a fractional duration is ignored without an error.** The chef "would not move" in the regression, though he moved fine by hand.
  - **The shift's own 3-second autosave overwrote the test's away time.**
  - **A storage write killed straight away is lost** (§5.11 again): the test now leaves with Home first.

## Commands
| Task | Command |
|---|---|
| Economy and rules | `npm test` |
| Desktop screenshot | `node tools/shot.mjs out.png "<js>"` |
| Art | `bash store/make-art.sh` |
| Debug APK | `JAVA_HOME=<path to JDK 21> npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Device | `node tools/device.mjs errors / eval / frames / shot`, `node tools/regress.mjs` |
| Signed bundle | `node tools/release.mjs` (cuts the `__dbg` hook with the autopilot and the money top-up) |

## Not done yet (by decision)
- Monetization: the spec's rewarded boosts, interstitials and IAP. Fair idle is a selling point, so any later "x2 with an ad" must be an extra, never a condition.
- More cities on the tour, daily quests, a drive-thru, and walking animations beyond a bob.
