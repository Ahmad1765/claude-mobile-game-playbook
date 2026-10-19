# Gemstack: Development Log and Retrospective

A complete record of how Gemstack was built: the research, what was built in each version, every major problem and how it was found and fixed, the current architecture, known gaps, and how I would build it differently if I started again.

Dates: 2026-10-17 to 2026-10-18. Test device: Infinix Hot 9 (X655C), MediaTek Helio P22, PowerVR GE8320 GPU, 4 GB RAM, 720×1600, Android 10, WebView 154.

---

## 1. Summary

| | |
|---|---|
| Product | **Gemstack: Block Puzzle**. An 8×8 block puzzle with an Adventure mode (gem-collecting levels), Classic (endless), and a Daily puzzle. |
| Business model | Free, earning from ads (AdMob): rewarded video first, capped interstitials, a bottom banner. |
| Tech | HTML5 Canvas + JavaScript modules, wrapped as a native Android app with Capacitor 8. No bundler and no framework. |
| Size | About 2,600 lines of source. The debug APK is 9.5 MB; the release AAB is 7.0 MB. |
| Cost so far | $0. Publishing needs the $25 Google Play fee. |
| Status | Feature-complete and running on a real low-end phone at about 60 fps. It is not published yet: real ad IDs, a privacy-policy URL, a signing key and the 14-day closed test are still needed (see `LAUNCH.md`). |

Supporting documents:
- `RESEARCH.md`: market research and the genre decision, with sources.
- `LAUNCH.md`: step-by-step publishing guide.
- `store/listing.md`: Play Store listing text and data-safety answers.

---

## 2. Timeline: what was done, in order

### Phase 0: Market research
**Goal:** pick a game that one person with no budget could realistically make money from.

**Findings** (full detail and sources in `RESEARCH.md`):
- Puzzle is the largest casual genre, and the only one in the casual top three that grew in 2025.
- Block Blast was the #1 most-downloaded game in the world from 2025 to 2026, earning an estimated ~$17M a month almost entirely from ads.
- Match-3 and merge games are dominated by big studios and need thousands of hand-made levels. Block puzzle needs **no level content**, because play is procedural and infinite.
- Hybrid-casual (simple gameplay plus progression) was the only casual segment whose in-app revenue grew. Rewarded ads bring about 62% of ad revenue and are opt-in.
- New personal Play developer accounts must run a **12-tester, 14-day closed test**.
- Honest revenue model: about $0.03–$0.08 per daily player on mostly organic Tier-2/3 traffic.

**Decision:** a block puzzle with the meta layer Block Blast lacks (coins, boosters, daily goals, themes), monetized through AdMob.

### Phase 1: v1, the minimum viable game
**Built:**
- The 8×8 board with 37 piece shapes, row and column clears, combos (which survive up to 3 placements without a clear), and a board-clear bonus.
- A **fair piece dealer**: at least one dealt piece always fits whenever any shape fits, and 35% of the time a line-clearing piece is offered. Difficulty ramps with score.
- A seeded daily challenge (same pieces for everyone), coins, Undo/Shuffle, 5 themes, a daily login reward, a rating prompt after a new best score, and auto-save.
- An AdMob wrapper with consent (UMP), a banner, frequency-capped interstitials and rewarded ads. In the browser everything is simulated, so the game can be played on desktop.
- Synthesized sound effects (no audio files) and haptics.
- The Capacitor Android project, launcher icons and a store feature graphic (generated with ImageMagick), privacy policy and listing text.
- Pure-logic tests.

**Problems found and fixed:**

| Problem | How it was found | Fix |
|---|---|---|
| Capacitor 8 needs JDK 21; the machine's default was JDK 17 | Gradle build error | Pointed Gradle at the JBR 21 already installed (`org.gradle.java.home`) |
| **The app didn't start at all on Android** | Installed on the phone; the home screen never rendered | Without a bundler, `Capacitor.registerPlugin` doesn't exist. Shipped core's prebuilt `capacitor.js` (`npm run copycore`) |
| The "Claim ×2" reward button was often hidden | Device screenshot: the rewarded ad took about 9 s to load | Buttons now show once ads are initialized, and `showReward()` waits up to 5 s for a load still in progress |
| The score display got stuck after Undo | Code review of the count-up formula | Handle a falling score separately |

### Phase 2: v2, gamification
The user's feedback: "way too simplistic."

**Added:**
- **Adventure mode:** procedurally generated, deterministic levels. Gems sit on pre-placed blocks that are mirrored left to right so levels look hand-designed. There are 1 to 3 gem types, 1–3 stars based on moves used against a par, and a map with 8 chapters of 20 levels each.
- **Player levels:** XP and titles from Rookie to Mythic, with a reward at every level-up.
- **Daily quests:** 3 per day, chosen deterministically from 10 types, plus a chest for finishing all three.
- **Achievements:** 12 of them, each with 5 tiers from Bronze to Diamond.
- **7-day login calendar** with a weekly chest, and a **lucky wheel** (one free spin a day plus 3 ad spins).
- **Booster inventory:** Undo, Shuffle, Hammer (smash one block) and Bomb (3×3 area). Hammer and Bomb have an aiming mode.
- A shop (boosters, a booster pack, free coins for watching ads) and 8 themes with 5 block styles.
- A first-game tutorial, procedural background music, local reminder notifications (daily reward, free spin, "we miss you"), and Android back-button handling.
- A save migration from v1 (single run slot → one slot per mode).
- Code split into modules: `logic`, `meta`, `store`, `fx`, `dom`, `themes`, `game`, `ui`.

**Problems found and fixed:**
- On the phone's shorter screen, the home layout overlapped. Fixed by restructuring the hero section and adding a short-screen layout.
- Wheel labels used a fixed radius. Fixed with a CSS variable.
- First-game text ("Best 0", a missing tutorial hint). Fixed.
- A forced Adventure win was tested end to end: stars, rewards, the level-up modal and the next level.

### Phase 3: v3, redesign and the first performance pass
The user's feedback: "the UI was not good and the game was extremely laggy."

**Measuring first.** I attached Chrome DevTools to the WebView on the phone (adb port forward plus Playwright `connectOverCDP`) and sampled frame times.

| Screen | v2 FPS | Late frames |
|---|---|---|
| Home | 60 | 0% |
| Game, nothing moving | **37** | 64% |
| Game, dragging | **37** | 63% |

JavaScript was under 3% of CPU time. Switching canvas features off one at a time on the live device showed the cost was **redrawing and re-uploading the whole canvas every frame**, done only for decorations: bobbing pieces, bobbing gems and a red "danger" pulse that used a blur shadow.

**Performance fixes:**
1. The canvas redraws only when something changes or an animation is running.
2. The board is cached as two images: the frame (rendered once per layout or theme) and the cells (re-rendered only when a move changes the board).
3. The piece under the finger became its own small canvas moved with a CSS `transform`, so dragging is a compositor update. The board repaints only when the snap cell changes.
4. The danger pulse became a CSS opacity animation. The background is static (no floating shapes).
5. Redundant DOM text writes are skipped. The device pixel ratio is capped at 2.
6. **A bug from v1 was found:** when the ad banner loaded, the canvas changed size but never re-measured, so the board was **squashed vertically and drops landed in the wrong place**. Fixed with a `ResizeObserver` on the canvas.

**Redesign** (using the frontend-design skill):
- **Diagnosis:** emoji icons rendered inconsistently on Android (the calendar emoji showed "July 17", the coin looked like a bank, the pause button was an orange box). There were generic purple and pink candy gradients, ALL-CAPS labels and floating background "smudges".
- **Direction: "jeweler's velvet tray".** A brass-framed board on teal velvet, with blocks as cut gems.
- **Tokens:** velvet `#0A242C`/`#134552`, brass `#F4B942`, cream `#FFF6E5`, ruby, emerald and sapphire accents.
- **Type:** Lilita One for display text and numbers; Fredoka for interface text; sentence case everywhere.
- **About 45 hand-drawn SVG icons** (`icons.js`): coloured objects with a dark outline, and line-style glyphs for menus. All emoji were removed from the UI.
- One button system (outlined "sticker" buttons), brass-rimmed modals and hero cards, and a frame colour for each theme.
- Fallbacks for older WebViews without `color-mix()`.

| After Phase 3 | FPS | Late frames |
|---|---|---|
| Home | 60 | 0% |
| Game, nothing moving | **60** | 0% |
| Game, dragging | **57** | 3% |

### Phase 4: effects performance
The user's feedback: "whenever the confetti appears the frame rate drops, or when the blocks are removed."

I added a test hook (`window.__dbg.clearTest`, `.confetti`) so effects could be triggered on demand while measuring.

| Effect (v3 build) | FPS |
|---|---|
| 1 line cleared | **16** |
| 3 lines cleared | **14** |
| Confetti alone | 48 |

This was found through bisection and full Chrome traces (the `Tracing` domain) on the device. Four causes, each confirmed by measurement:

| Cause | Evidence | Fix |
|---|---|---|
| The score counted up every frame, and the best-score bar animated its width, repainting the HUD over the gradient background | Hiding the HUD doubled the frame rate | The score updates at once with a compositor-only "pop"; the bar uses `transform: scaleX`; the background is on its own layer (`translateZ(0)`) |
| The confetti loop read `innerWidth`/`innerHeight` for every piece, every frame | The trace showed about 4,600 forced style/layout passes in 1.6 s, all from `loop` in `fx.js` | Read the dimensions once per burst |
| The full-screen confetti canvas re-rasterized the whole page every frame | The trace showed the document painted 52 times in the window | In-game confetti is drawn on the game's effects canvas; menu screens use a small DOM burst |
| A single canvas did all the drawing; screen shake translated the canvas | Hiding the effects layer → 57 fps | Two stacked canvases (static board+tray; effects); shake is a CSS transform; praise text is pre-rendered sprites; the save is deferred off the drop frame |

A dead end, recorded for honesty: creating 36 WAAPI-animated DOM confetti pieces at once cost 110–180 ms (one GPU layer each), which was worse. Lowering the effects canvas resolution made no difference, which ruled out pixel fill as the cause.

| Final, v4 build | FPS | Late frames |
|---|---|---|
| Nothing moving | 60 | 0% |
| Confetti alone | 60 | 1% |
| 1 line cleared | 56 | 3% |
| 3 lines cleared | 56 | 8% |
| Board clear + confetti | 56 | 7% |
| Dragging | 57 | 3% |

The one remaining hitch: the single frame where a block lands can take about 50 ms (rebuilding the cell layer, audio and haptics calls, HUD updates).

---

## 3. Architecture (current)

```
www/
  index.html   screens + modals markup, all CSS (design tokens, components)
  icons.js     SVG sprite + ic(name); swaps <i data-ic> placeholders at boot (imported first)
  logic.js     pure rules: shapes, place/clear, scoring, fair dealer, seeded RNG, adventure level generator
  meta.js      pure progression tables: boosters, XP curve, quests, achievements, wheel, calendar, chapters
  store.js     save data (with v1 migration), stats tracking (bump/maxStat), grant/spend, event bus
  themes.js    8 themes (palette, block style, frame metal) + setTheme()
  fx.js        synthesized SFX + music, haptics, confetti (in-game sink / DOM fallback), coin fly, toast
  dom.js       $(), modal stack, screen router, small template helpers, setText()
  ads.js       AdMob wrapper (consent, banner, interstitial cap, rewarded); simulated in browser
  game.js      board engine: layered rendering, input, moves, boosters, modes, results, tutorial, __dbg
  ui.js        entry: home, map, quests, shop, trophies, calendar, wheel, level-ups, back button, notifications
test/
  logic.test.mjs  placement, clears, scoring, fair deal, revive, determinism
  meta.test.mjs   300 adventure levels (deterministic, gems == goals, mirrored, no full lines), XP, quests, tiers, wheel
store/          icons, feature graphic, listing, privacy policy, make-icons.sh
android/        Capacitor native project (manifest has AdMob App ID, portrait, signing from keystore.properties)
```

**Rendering pipeline (game screen):**
- `frameLayer`: brass frame, blurred shadow, velvet well and empty cells. Rendered on layout or theme change.
- `layer`: `frameLayer` plus blocks and gems. Rendered when the board changes.
- `#cv` (static canvas): `layer` plus the tray. Repaints when its signature changes (board, pieces, dragged slot, targeting).
- `#ov` (effects canvas): ghost piece, line highlights, pops, clear animations, particles, praise sprites, confetti. Repaints only while something is moving; hidden when empty.
- Drag piece: a separate small canvas moved with `translate3d`.
- Shake: a `transform` on `#stage`.

**Data flow:**
1. A move calls `logic` to work out the result.
2. `game` applies it, calls `store.bump(...)` for stats and updates the HUD.
3. `store.persist()` emits `change`, which refreshes the red-dot badges.
4. XP crossing a threshold emits `levelup`, which `ui` turns into the level-up modal queue.

---

## 4. Known gaps and risks (as of this log)

**Must do before publishing:**
- Real AdMob IDs in `www/ads.js` and `AndroidManifest.xml` (still Google's test IDs).
- Privacy-policy URL (placeholder; the support email is now filled in), signing key, then the 12-tester closed test.
- Decide the final name and package id. "Gemstack" has not had a trademark search; the package id can never change after the first upload.

**Should do:**
- No analytics or remote config. Retention (D1/D7) and economy can't be measured or tuned without an update.
- Adventure difficulty, par values and the coin economy have **never been tested with real players or simulated**.
- No "Remove ads" purchase and no leaderboards (Play Games Services).
- In-game text is English only (only the store listing is planned for translation).
- `android/gradle.properties` contains a machine-specific JDK path.
- `window.__dbg` ships in production. It's harmless (single-player, local data), but it could be removed for release.
- Launcher icon and splash colour still use the v1 navy palette rather than the v3 velvet/brass palette.

**Unverified:**
- The EEA consent form (tested only in a "not required" region).
- Whether notifications actually fire.
- Music audibly.
- Behaviour on very old WebViews (below Chrome ~99, which lacks `roundRect`).

---

## 5. Retrospective: how I would build it from scratch

The final game is good, but it took four passes to get there. Most of the rework came from decisions made too late: the target device, the art direction and the rendering model. Here's what I'd change.

### 5.1 Lock in the hard constraints on day one
1. **Pick the weakest target phone first and set a performance budget.** For example: 60 fps when idle and at least 55 fps during effects, on a Helio P22-class device. I tested on the phone from v1, but only for "does it work", not "is it smooth". The lag in v2 and v3 would have been caught the day it was introduced.
2. **Build the measurement harness before features.** The CDP frame sampler, the trace summarizer and the `__dbg` effect triggers were the most valuable tools in the project, and they arrived in Phase 3. Next time they'd exist in week one and run after every visual change.
3. **Decide permanent things first:** the name (with a trademark search), the package id, the signing key, and a portable JDK setup (`JAVA_HOME`, not a hard-coded path).

### 5.2 Technology choice, reconsidered
I chose Capacitor + Canvas 2D because it gave very fast iteration and let me screenshot and drive the game with Playwright. That paid off. With hindsight I'd weigh it differently:

| Option | For | Against |
|---|---|---|
| **Capacitor + Canvas 2D** (used) | Tiny APK; instant reload; browser automation | Performance depends on the phone's WebView and GPU; easy to fall into DOM-repaint traps; AdMob plugin is community-maintained |
| Capacitor + **PixiJS (WebGL)** | Same workflow, but GPU-batched sprites avoid most canvas upload and repaint costs | Still a WebView; another dependency |
| **Flutter + Flame** (Flutter already installed here) | Native rendering, consistent across devices; **official** Google plugins for ads, in-app purchases and Firebase; this game is about 70% menu UI, which is Flutter's strength | Slower visual iteration; bigger APK (~15–20 MB) |
| Godot 4 | Excellent 2D engine, free | Weaker UI tooling for menu-heavy games; harder to automate tests |

**What I'd pick:** for a menu-heavy hybrid-casual game aimed at low-end Android, **Flutter (+ Flame for the board)**. It removes the "WebView lottery" and gives first-party ads, purchases and analytics. If I stayed on the web stack, I'd use **PixiJS from day one**, plus a small declarative view layer (Preact or lit-html) for the menus instead of hand-built `innerHTML` strings.

### 5.3 Design system before features
- v2 shipped about 15 screens built with emoji and ad-hoc gradients, and v3 had to rewrite every template. Next time, the art direction, tokens, icon sprite and component kit (buttons, panels, plates, pills, rows) come **first**. Every feature screen is then assembled from that kit.
- **No emoji in the UI, ever.** They are fonts that differ by Android version and maker.
- Design for the short screen first (360×620 once the banner is there), not a tall desktop viewport.

### 5.4 Rendering rules from the start
Everything learned the hard way in Phases 3–4, applied up front:
- Redraw only when something changes. Use layered canvases (static vs. effects) and cache anything that doesn't move.
- Moving things should be compositor transforms (drag piece, shake, score pop). Never animate `width`, `top` or text every frame.
- Never read layout (`innerWidth`, `getBoundingClientRect`) inside a frame loop. Read once and cache.
- No full-screen canvases or blur shadows per frame. Pre-render text and shadows into sprites.
- Account for size changes the window doesn't announce (banner ads) with `ResizeObserver`.
- Defer saves and other bookkeeping off the frame where a block lands.

### 5.5 Architecture
- **Keep:** pure, tested `logic.js` and `meta.js`. This was the best decision in the codebase: every refactor of rendering and UI left the rules untouched and still passing tests.
- **Change:** use domain events instead of scattered stat calls. The game would emit `piecePlaced`, `linesCleared`, `gemCollected`, `runEnded`, and quests, achievements and XP would subscribe. Today `game.js` calls `bump()` and `maxStat()` directly, which couples gameplay to progression.
- **Change:** use declarative screens with a small reactive store, instead of `showScreen('quests')` re-renders, ID lookups and manual `setText` guards.
- **Change:** have one source of truth for board geometry, shared by the renderer and the test bot. The bot duplicated the layout formula and broke whenever it changed.
- **Change:** have tests check game state directly rather than through save data. The bot read `localStorage`, so deferring saves made it miscount moves.

### 5.6 Balancing with simulation, not guesswork
- A headless bot that plays 10,000 Classic games to measure score distribution, game length and how often players are stuck. That would tune `shapeWeight` and the dealer's helpfulness with data.
- A solver that plays every Adventure level many times, to check each level is winnable and that par (the 3-star threshold) is reachable but challenging.
- An economy model (coins earned per day against booster and theme prices and ad watches per session), so prices are set deliberately.

### 5.7 Live-ops plumbing on day one
- Firebase Analytics, Crashlytics and Remote Config (all free). Difficulty, ad frequency and prices would then be adjustable without an app update, and A/B testable.
- In-app purchase code ("Remove ads") present but switched off until there are about 1,000 daily players.

### 5.8 Scope and sequencing
- v2 added seven retention systems in one pass. Four of them are "come back daily" mechanics: calendar, wheel, quests and the daily puzzle. That's a lot for a new player to parse. Next time I'd ship them in order of expected retention impact: **game feel → Adventure → daily puzzle → quests**, and add the wheel and calendar only if the data shows a need.
- Smaller vertical slices, each validated on the phone for both function and frame rate before moving on.

### 5.9 Reuse before building (added 2026-10-19, during game 01)
Never start a new game or feature from scratch. In order: reuse what is in this repo (Gemstack's kit, the device and test tools), then what the platform or installed libraries already provide (Capacitor plugins, cached Playwright/Chromium, ImageMagick, Fontsource), and only then write new code, as little as possible. See `lotus/DEVLOG.md`.

### 5.10 Process lessons (my own mistakes)
- Writing patches as complex `sed`/heredoc one-liners broke twice: once with a nested-quote shell error, and once when a slice boundary deleted the line that starts the render loop. Both were caught by the browser regression run within minutes. Writing patches to files was more reliable, and running the full regression after every change was essential.
- My first effects measurements were confounded: the test board was otherwise empty, so every "line clear" was really a "board clear with confetti". Isolate one variable per measurement.
- Test on the real device with the **real ad banner loaded**. The squashed-board bug lived from v1 to v3 because desktop tests had no banner.
- (Added 2026-10-20, game 02.) **Empty stray files** named after code fragments (`{let`, `!g)`, `setTimeout(r`) appear in project folders, including `www/`, where they would ship in the APK. A hook in this Claude Code setup passes tool input through `cmd.exe`, which treats `>` (from `=>` in code) as a redirect. Each game's `tools/release.mjs` should refuse to build with empty files in `www/` (Word Passport's does).
- (Added 2026-10-20.) Don't patch files with Python on Windows: text mode writes CRLF, which broke Word Passport's release-script pattern. Use the Edit tool or Node.
- (Added 2026-10-20.) Check the game name against store search before writing anything: spec 02's "Word Trail" was already taken by two apps.

---

## 6. How to resume work quickly

| Task | Command |
|---|---|
| Play in the browser | `npm run serve` → http://localhost:8080 |
| Logic tests | `npm test` |
| Debug APK onto the phone | `npm run apk`, then `adb install -r android/app/build/outputs/apk/debug/app-debug.apk` |
| Release bundle | `npm run aab` (needs `android/keystore.properties`, see `LAUNCH.md`) |
| Profile on the phone | `adb forward tcp:9223 localabstract:webview_devtools_remote_<pid>`, then connect Playwright `connectOverCDP('http://localhost:9223')`. Trigger effects with `__dbg.clearTest(rows, emptyBoard)` and `__dbg.confetti(n)` |
| Tune the game | `logic.js` (difficulty, dealer, level generator); `meta.js` (economy, quests, achievements, wheel, calendar); `themes.js` (prices, level locks) |
