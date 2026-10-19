---
name: mobile-game-dev
description: Rules for building casual mobile games (especially HTML5/Canvas in a WebView via Capacitor, targeting low-end Android) that run smoothly, ship cleanly and don't need rework. Use when starting a new mobile game, adding game features, fixing lag or jank, preparing a Play Store release, or testing a game on a real phone.
---

# Mobile game development playbook

These rules come from building three casual games for Android with Claude Code (a block puzzle, a mahjong game and a word search), all tested on a low-end phone (MediaTek Helio P22, 4 GB RAM, Android 10). Each rule is here because breaking it cost a rewrite, a bug or a lost day. The case studies in this repo have the measurements behind them.

Follow them in order. When a rule doesn't fit the project, say so and why, rather than skipping it silently.

## 1. Day one: decide the permanent things before writing code

- **Check the name in the stores first.** Search Google Play and the App Store for the exact title and close variants. One planned title ("Word Trail") was already used by two apps; finding that after the icon, listing and package id exist means redoing all three. Never use a competitor's trademark.
- **Fix the package id / bundle id now.** It can never change after the first upload. Use one namespace per game (`app.<gamename>.game`) so a game can be sold or moved on its own.
- **Create the signing (upload) key now, one per game,** keep it and its passwords out of version control, and tell the user to back both up off the machine. Enrol in Play App Signing.
- **Pick the weakest phone you will support and set a performance budget,** for example: 60 fps idle and at least 55 fps during effects on a Helio P22-class phone. "It runs" is not the bar; "it runs smoothly on the weak phone" is.
- **Build the measurement tools before the features** (section 5). In the first game they arrived in the third rewrite; they were the most valuable code in the project.
- **Keep the toolchain portable.** Point builds at the JDK with `JAVA_HOME` at build time, not a machine-specific path committed in `gradle.properties`.

## 2. Reuse before building

Never start a game or feature from scratch. In order:
1. Code and tools already in this repo or the previous game (copy the whole last project and trim it; don't rebuild its save system, sound, screens or device tools).
2. What the platform and installed libraries already provide (Capacitor plugins, system fonts packages, ImageMagick, Playwright's browser).
3. Only then new code, as little as possible, for the part that is missing or broken. When a reused part misbehaves, replace only that part.

At the start of each new game, list which files and tools it reuses before writing anything. In the third game, copying the second one meant the Android setup, saves, sound and device tools worked on the first build, and nearly all new work was the game itself.

## 3. Design system before screens

- **Art direction, colour tokens, an icon set and a small component kit (buttons, cards, modals, rows) come first.** Every screen is then assembled from the kit. The first game shipped about 15 screens built ad hoc and had to rewrite every one.
- **No emoji in the UI, ever.** They are fonts that differ by Android version and phone maker (a calendar emoji showed a date, a coin looked like a bank). Use an inline SVG icon set.
- **Design for the short screen first** (about 360×620 CSS px once a banner ad is there), not a tall desktop viewport.
- **Check the phone's system text size at 1.15× and 1.3×** before calling any screen done. WebView scales CSS text with it (canvas text is not scaled), so lists grow, titles wrap and the game area shrinks. This found layout bugs in both of the last two games that desktop testing never showed.
- **Messages and banners must not resize the play area.** Show helper text as an overlay; a message that pushes the board up makes it jump under the player's finger.
- **Preview art at the size it will really be seen,** and against what will be behind it. An icon detail that reads at 1024 px can hide a letter at 48 px; a navy stamp disappears on a navy cover.

## 4. Rendering rules for Canvas in a WebView

On a weak GPU, JavaScript was under 3% of CPU time; the lag was all painting and uploading pixels.
- **Redraw only when something changes.** No permanent `requestAnimationFrame` loop. Use a dirty flag and draw once per change.
- **Layer it:** a static canvas for the board (cache anything that doesn't move, such as the frame, card or letters, in offscreen canvases), and a separate effects canvas that animates only while something moves and is hidden when empty.
- **Moving things are compositor transforms:** a dragged piece is its own small element moved with `translate3d`; screen shake and "pop" effects are CSS transforms. Never animate `width`, `top`, `left` or text content every frame.
- **Never read layout inside a frame loop or a pointer-move handler** (`getBoundingClientRect`, `innerWidth`, `offsetWidth`). Read once per layout, or once per gesture, and cache. One confetti loop caused about 70 forced layouts per frame.
- **No full-screen animated canvases, no `shadowBlur` per frame.** Bake shadows and text into sprites once. A full-screen confetti canvas re-rasterised the whole page every frame.
- **Don't create dozens of animated DOM elements at once** (36 animated confetti divs cost 110–180 ms in one frame: one GPU layer each).
- **Watch for size changes the window doesn't announce** (an ad banner loading) with a `ResizeObserver` on the play area. Missing this squashed the board and made drops land in the wrong place for three versions.
- **Cap devicePixelRatio at 2.**
- **Defer saves and bookkeeping off the frame where the player's action lands** (save about 500 ms later, and immediately when the app goes to the background).
- **Effects can outlive the state they draw.** A 300 ms animation still running after the player quits must not read game state that is now gone. Keep geometry the effects need outside the game-state object, and guard input handlers against a quit mid-gesture.
- **Time the work that runs when a screen opens, not just animations.** Generating a puzzle took 210 ms at worst on desktop, which is a visible freeze on a weak phone; one fix made it 8× faster. Measure generation and layout, not only fps.

## 5. Measure on the real phone

- **Desktop Chrome is not evidence.** Attach to the game's WebView on the phone over USB: `adb forward tcp:9223 localabstract:webview_devtools_remote_<pid>`, then Playwright `chromium.connectOverCDP('http://localhost:9223')`. A small script can then evaluate JS in the page, sample `requestAnimationFrame` intervals (fps, % of frames over 20 ms, worst frame) while an action runs, and take screenshots (`adb exec-out screencap -p`). See `tools/device.mjs` in this repo.
- **Add a debug hook (`window.__dbg`) that triggers each effect on demand** (a line clear, a win, confetti, a hint) so each can be measured alone. Strip it from release builds with a script that fails loudly if it can't find it.
- **Change one variable per measurement.** One early test board was empty, so every "line clear" was really a "board clear with confetti", and the numbers pointed the wrong way.
- **Find causes with traces, not guesses:** switch features off one at a time on the device, and use the Chrome `Tracing` domain to count forced layouts and paints.
- **Test with the real ad banner loaded** once ads exist. Desktop tests have no banner.

## 6. Architecture

- **Keep the rules pure and tested** (`logic.js`: no DOM, deterministic, seeded random numbers). Every rewrite of rendering and UI in three games left the rules untouched and their tests still passing.
- **One source of truth for geometry,** used by the renderer, the hit test, the effects and the test bot. A bot that copied the layout formula broke every time the layout changed.
- **Tests read game state, not saved data.** Deferring saves made a bot that read `localStorage` miscount.
- **Record wins and unlocks the moment they happen,** not after the celebration; a player who quits during the confetti must not lose the level.
- **Separate save slots per mode** (journey vs. daily), so starting one never erases the other.
- **Validate generated content automatically.** For procedural levels: thousands of seeds, each checked for solvability and fairness (every level solvable; every hidden word readable in exactly one place; no offensive words in random filler). For written content: spell-check every word against a dictionary and fail on typos. A spell-check of 2,290 hand-written words caught 8.
- **Plan for content updates after launch.** If puzzles are derived from content lists by shuffling, adding items reshuffles every existing puzzle. Use an order that stays stable when items are appended, and guard saved in-progress state against content that has changed under it.

## 7. Test like a player, on the device, after every change

- **Keep an on-device regression script** and run it after every change: it drives the running app over CDP, uses real touch input where it matters (`adb shell input swipe/tap` at coordinates from the engine's own geometry), and asserts game state.
- **Fail the run on any page error,** not only on failed assertions. That check caught a crash manual play would almost never hit.
- **Relaunch the way a person leaves:** press Home, then kill the process. On Android WebView, `localStorage` written less than about 1 second before a hard kill in the foreground can be lost; going to the background flushes it.
- **Write down where every wait in a test comes from** ("700 ms celebration + 1300 ms petals"). A guessed wait of 1.9 s for a card that opens at 2.0 s made a correct game look broken.
- **Check full-screen / edge-to-edge on the real phone.** On Android 10, Capacitor 8's system-bars inset listener reported a 0 px bottom inset, putting buttons under the navigation bar; the fix was one replacement inset listener in `MainActivity`.

## 8. Game design and scope

- **Ship in slices, each checked on the phone for function and frame rate:** game feel first, then progression (journey/levels), then a daily puzzle, then everything else. One version added seven retention systems at once (four of them "come back daily"), which was too much for a new player to parse.
- **Design the reward loop before ads exist** (what hints cost, how they're earned) so rewarded ads plug in later without changing the rules. Close farming loopholes (restarting a level must not re-earn rewards).
- **Offer an alternative to the hard gesture** (tap first and last letter as well as swiping; tap to select as well as drag). Many players, especially older ones, struggle with precise swipes.
- **Never let a player get permanently stuck** (a shuffle that always succeeds, a guaranteed-solvable deal).
- **Get real players early.** Difficulty and economy tuned by judgement alone are guesses; a few closed-test players are worth more than more polish.

## 9. Working habits for Claude Code

- **Make code changes with the editor tool or small script files, not long `sed`/heredoc one-liners.** One-liners broke twice in the first game (a quoting error, and a slice that deleted the line starting the render loop).
- **On Windows, don't patch files with Python text-mode writes:** they convert line endings to CRLF, which broke a release script's pattern match. Use the editor tool or Node.
- **Watch for empty stray files named after code fragments** (`{let`, `!g)`, `setTimeout(r`). In one setup a hook passed tool input through `cmd.exe`, which treated `>` (from `=>` in code) as a redirect. In a web game's asset folder they ship inside the app. Make the release script refuse to build if the asset folder contains empty files, and delete them when found.
- **Release scripts should fail loudly** when something they expect is missing (the debug hook to strip, the signing config). Two such checks caught real problems.
- **Verify every number before writing it into docs** (counts, sizes, timings); re-run the command rather than recalling it.
- **Keep a DEVLOG per game:** decisions, measurements, bugs found and how, process problems with the guard added for each, commands, and what is deliberately not done yet. Read the previous game's DEVLOG before starting the next one.
