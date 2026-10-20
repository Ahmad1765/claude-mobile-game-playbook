# Lotus Mahjong: Development Log

Game 01 of the portfolio (`../games/01-lotus-mahjong.md`). It's a calm mahjong pair-matching game for players aged 45 and over. Ads and other monetization are deliberately not built yet. Test device: Infinix X655C (Helio P22, Android 10, WebView 154, **system font scale 1.15**).


## v2 (2026-10-20): Japandi & wabi-sabi, the lotus pond, clear-view tiles, a breathing break
After the owner's feedback (root DEVLOG §5.16), restyled from scratch in a style from the owner's list and given three approved selling points.
- **Style: Japandi & wabi-sabi.** Washi paper, sumi ink, oak and clay; flat paper with hairline edges, no shadows on the UI, square-ish corners, a left-aligned title with a small clay seal, a home menu of plain ruled lines. Imperfection on purpose: a brushed ensō, a pond that isn't an oval. Tiles are oak-backed bone with muted clay, moss and indigo ink. Shippori Mincho for titles, Zen Kaku Gothic New for text (Fontsource, OFL). Store art: the lotus tile inside the ensō.
- **The lotus pond** (`pond.js`): home shows a pond with one lotus leaf per board cleared (`stats.wins`); every third leaf flowers, and once the pond holds 48 leaves one more flowers per board. Leaves you haven't seen yet grow in (at most the last 8, so it never drags). The win card says "A lotus opened in your pond".
- **Clear-view tiles:** an eye button in the HUD (and a setting). Tiles get as big as the screen allows (padding 4, cap 140 instead of 88), each suit gets its own pale face colour, outlines go dark, corner numbers turn on, blocked tiles fade far back, and with a tile selected every free tile it can match is outlined, free (no star cost). Clearing a level in clear view still earns 3 stars.
- **A breathing break** (`breath.js`): "One-minute breath" on the win card and in the pause menu. In for 4 seconds, out for 6, six times; the ensō scales (compositor only), a small synthesized singing bowl sounds at each breath in, a light haptic marks each turn. Android back ends it and its timers.
- **Outside resources:** Kenney "Casino Audio" (CC0) chip clacks for tiles and a card shuffle trimmed to 1 s; Fontsource fonts. Nunito removed.
- **Measured on the phone (v2):** idle 60 fps; four matches 450 ms apart 59.5 fps; the breathing ensō 60 fps. Every screen fits at 1.3× text (one button label shortened to "One-minute breath"). Regression 9/9, including a real tap in clear view.
- **Notes:** the phone had a release-signed copy, so the debug build would not install over it (`INSTALL_FAILED_UPDATE_INCOMPATIBLE`); uninstalling cleared the test save. Store screenshots retaken in the new style.

## Rule: build on what already exists, never start from scratch
Before writing anything new, check what is already available and reuse it. That saves time, and the reused parts are already tested on the device:
1. **This repo first.** Gemstack's kit (`../www/`: store, fx/sound, dom helpers, icons pattern, ads wrapper, back button, notifications) and the tools here (`tools/device.mjs`, `regress.mjs`, `sim.mjs`, `shot.mjs`). Copy and trim instead of rewriting.
2. **Then the platform and the libraries already installed.** For example, Capacitor's built-in `SystemBars` plugin, the `App`/`Haptics` plugins, Fontsource fonts in `node_modules`, Playwright's cached Chromium, and ImageMagick for icons.
3. **Only then write new code, as little as possible,** and only for the part that is missing or broken. The edge-to-edge fix below is the example: Capacitor's plugin does the bar styling, and we replaced only the one listener that misbehaves on Android 10.

Every new game in `../games/` starts by listing which existing files and tools it will reuse.

## How the Gemstack lessons were applied

| Gemstack lesson (`../DEVLOG.md`) | What happened here |
|---|---|
| Set a perf budget and build the measurement harness before features | `tools/device.mjs` (CDP frame sampler, eval, screenshots) existed before the first screen was finished. Budget: 60 fps idle, ≥55 fps during effects |
| Design system first; no emoji | Tokens and a tile motif (ivory face over a jade edge) are used by every card and button. Icons are SVG glyphs only |
| Design for the short screen | All checks ran at 360 px wide, including at **1.3× system font size** |
| Redraw only on change; compositor-only motion | The board canvas redraws only on taps. The matched pair is two small DOM canvases moved with transforms. Win petals run on an effects canvas that is hidden when empty |
| Account for resizes the window doesn't announce | `ResizeObserver` on the stage. The helper message is an overlay, so it never resizes the board |
| Defer saves off the frame | Saves wait 500 ms, and the app saves when it goes to the background |
| One geometry source for renderer and bots | `__dbg.tileCenter/tap/state` use the engine's own geometry |
| Tests read game state, not save data | `tools/regress.mjs` asserts `__dbg.state()` |
| Balance by simulation | `tools/sim.mjs` (random and careful players per level band) |
| Patches via files, not shell one-liners | Followed after one `package.json` escaping slip (caught immediately) |
| Portable JDK | No JDK path in `gradle.properties`. Set `JAVA_HOME` to JDK 21 when building |
| Scope: game feel → journey → daily first | Built only those, plus settings. No wheel, calendar rewards, quests or coins |

## What exists
- **Rules (`logic.js`, pure).** Free-tile rules, a **reverse-solved deal** (every board is provably solvable), a face shuffle that keeps the board solvable, `rescue()` (if a shuffle can't help, it returns pairs to the board, so a player can never be trapped), a DFS solver for hints, 11 hand-made shapes, mirrored procedural layouts, and the difficulty ramp.
- **Tiles (`tiles.js`).** All 42 faces are drawn with canvas paths plus a bundled 3 KB CJK glyph subset (Noto Serif TC, OFL, 16 characters), so rendering doesn't depend on the device's fonts. Optional corner numbers help players who don't read Chinese numerals.
- **Game (`game.js`).** Selection, matching, a stack shadow so layers read apart, shading for blocked tiles, Hint / Undo / Shuffle, a stuck dialog, a 2-step tutorial on level 1, stars (3 = no hints or shuffles), and a petal celebration.
- **Meta (`app.js`).** Home, a journey map (gardens of 15 levels that "bloom"), a daily puzzle with a month stamp calendar, settings (sound, music, vibration, shade blocked tiles, tile numbers), the Android back button, and a branded splash and launcher icon.

## Measurements (on the phone)
| Scenario | FPS | Late frames |
|---|---|---|
| Idle, level 150 board | 60 | 0% |
| 6 rapid matches | 59.7 | 0.5% (one frame) |
| Shuffle / hint / blocked-tap feedback | 60 | 0% |
| Last match + win celebration | 60 | 0% |
| New board (sprites built) | one 50–80 ms frame during the screen change | |

Tile size: 52–54 px on 268 of the first 300 levels, and never below 52 px (layouts are capped at 6×7 tiles for portrait phones).

Simulated dead-end rate (needs a shuffle). Levels 1–10: 0% for both a random and a careful player, since only unique pairs are used. From level 41 on: about 35–45% for a random player and 8–14% for a careful one.

## Bugs found by testing on the device (would have shipped from desktop testing)
- The system font scale (1.15) wrapped the title and the helper text. The helper text resized the stage, which made the board jump.
- The calendar grid overflowed at large text (`aspect-ratio` cells inflate `1fr` columns). Fixed with `minmax(0,1fr)` and fixed-height cells.
- The settings captions sat beside their labels (`.row span` also matched the nested span).
- On cold start, the shell painted before the fonts and icons were ready, showing empty buttons. The app now stays hidden until boot finishes. The default Capacitor splash and icon were replaced.
- **localStorage durability:** a hard kill less than about a second after a write loses it. Going to the background flushes it (measured both ways). Exposure is limited to a foreground crash losing under a second of progress.

## Issue: the game didn't use the full screen (notch / status bar area) — fixed 2026-10-19
**Symptom.** On the phone the app sat between an opaque status bar and an opaque navigation bar. The notch area at the top was empty instead of showing the game's background.

**Cause.** On Android 10 the window isn't edge-to-edge by default, so the system reserves the bar and notch areas. When edge-to-edge was switched on, Capacitor 8's `SystemBars` inset listener reported a **0 px bottom inset**. Its "is the keyboard open?" check misfires on Android 10, so the buttons ended up under the navigation bar. This was measured: Android reports top 80 px (the notch) and bottom 96 px; Capacitor passed 40 and 0 dp to the page.

**Fix (reusing what exists).**
- `MainActivity.java`: edge-to-edge window, transparent bars, and `LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES` so the app draws into the notch.
- One inset listener passes Android's real values to the page as `--inset-top/bottom/left/right`. They're resent on page load through Capacitor's `WebViewListener`.
- `capacitor.config.json`: `SystemBars` keeps the bar icon style (`LIGHT` = dark icons) with `insetsHandling: "disable"`.
- `index.html`: `#app` is inset by `max(env(safe-area-inset-*), var(--inset-*))`. The background and modal dimming fill the whole screen; buttons and text stay inside the safe area.

**Status bar hidden (follow-up, same day).** At the owner's request the status bar (clock, battery, signal) is now hidden completely. A swipe down from the top shows it briefly. `MainActivity.onWindowFocusChanged` hides it with AndroidX `WindowInsetsControllerCompat`, which was already a dependency. It runs on every focus gain because Capacitor's `SystemBars` re-shows the bars at load and Android restores them on resume (both checked on the device). The navigation bar stays visible. Capacitor's own `SystemBars.hidden` option wasn't used because it hides both bars. The top 40 px stays empty on purpose: the test phone's camera hole is in the top-left corner, right where the pause button is, and Android still reports it as an inset.

**Navigation bar hidden too (follow-up, same day).** At the owner's request the game is now fully immersive: both the status bar and the navigation bar are hidden, using the same line with `systemBars()` instead of `statusBars()`. A swipe from the top or bottom edge shows the bars briefly, so Back and Home stay reachable, and the hardware Back gesture still opens Pause (checked on the device). With the navigation bar hidden, Android reports a 0 px bottom inset, so the game now uses the full height down to the bottom edge (content runs from 40 to 800 px). Keep in mind for the older audience: some players may not know the edge swipe. If testers get lost, show the navigation bar again by changing that one line back to `statusBars()`.

**Verified on the device:** the WebView is 800 px tall (full screen, before the fix it was 712), the content runs from 40 to 752 px, and the regression passes.

**Watch for:** when the AdMob banner is added, keep `--banner` *plus* the bottom inset. Re-check on a phone with gesture navigation and a punch-hole camera, and in landscape if that's ever allowed.

## Bugs found by code review
- The win was saved 650 ms after the last match, so quitting during the celebration could lose the level unlock. It's now recorded on the match.
- A stale "stuck" timer could fire after Undo or on a new board. Timers now check that the board is the same one and still stuck.
- Two identical tiles stacked on top of each other can't be fixed by any face shuffle. Without `rescue()` the player would be trapped for good.
- A single run slot meant starting a journey level erased an in-progress daily. There are now separate slots per mode.

## Commands
| Task | Command |
|---|---|
| Logic tests (1,000 levels, 400 dailies, solver, shuffle, rescue) | `npm test` |
| Difficulty simulation | `npm run sim` |
| Desktop screenshot | `node tools/shot.mjs index.html out.png 360 700` |
| Tile art sheet / icon art | `node tools/shot.mjs "sheet.html?w=46" out.png`; `?art=icon|round|fg|splash` |
| Debug APK onto the phone | `set JAVA_HOME=<JDK 21>` then `npm run apk`, `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Signed Play bundle (strips `__dbg` and `sheet.html` from the packaged copy) | `set JAVA_HOME=<JDK 21>` then `npm run release` → `android/app/build/outputs/bundle/release/app-release.aab` |
| Device: errors / eval / FPS | `node tools/device.mjs errors`, `eval "<js>"`, `fps "<js>" [ms]` |
| Device regression (run after every change) | `node tools/regress.mjs` |

## Release setup (2026-10-19)
- **Package id is final: `app.lotusmahjong.game`.** Reverse-domain of lotusmahjong.app, which was unregistered at the time (lotusmahjong.com is someone else's store). Free on Play; no exact "Lotus Mahjong" title exists there, only "Mahjong Lotus Solitaire" (pm4), so the listing title is "Lotus Mahjong: Big Tiles". A per-game namespace keeps each game separable if one is ever sold.
- **Upload key:** `android/lotus-upload.jks` with its random password in `android/keystore.properties` (both gitignored). **Back up both off this machine.** Enrol in Play App Signing, so a lost upload key can be reset through Google.
- `tools/release.mjs` strips `__dbg` and `sheet.html` from the packaged copy only, so debug APKs still work with `regress.mjs`. It fails loudly if the `__dbg` block moves.
- Store listing, privacy policy and Play Console answers: `store/listing.md`, `store/privacy.html` (hosted at https://omitt.me/app-policies/lotus-mahjong/privacy.html from the GitHub repo Ahmad1765/app-policies), `store/feature-1024x500.png`, `store/screens/`.
- **Verified on the phone (2026-10-19):** the debug build with the new id passes `regress.mjs`. The signed release APK (built from the same stripped assets as the AAB, without the DEBUGGABLE flag) installs, starts full screen, runs the level 1 tutorial and matches a pair, with no crashes in logcat. The first install of a newly signed APK hung for about 3 minutes, which was Play Protect scanning it with no prompt on screen. adb isn't on the PATH: use `%LOCALAPPDATA%/Android/Sdk/platform-tools/adb.exe`, as `tools/device.mjs` does.

## Not done yet (by decision)
- **Monetization:** ads.js, the banner (re-test layout with the real banner loaded, see Gemstack §5.9), "Remove ads".
- Zoom and pan for large boards (100+ tiles), tile sets/themes, brain stats, reminders, analytics. Add these in that order only if players ask or the data shows a need.
- Unverified: music audibility, and behaviour on WebViews older than about Chrome 99 (`roundRect`).
