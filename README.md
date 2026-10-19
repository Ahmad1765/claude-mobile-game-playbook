# Claude Code mobile game playbook

Rules that make Claude Code build better casual mobile games, learned the hard way across three Android games built with it. They're packaged as a **Claude Code skill**, so Claude follows them automatically whenever you work on a mobile game.

The games: a block puzzle, a mahjong game and a word search, all made with HTML5 Canvas inside a Capacitor WebView and tested on a low-end phone (MediaTek Helio P22, 4 GB RAM, Android 10). Most rules apply to any mobile game; section 4 is specific to Canvas in a WebView.

## What's inside
| Path | What it is |
|---|---|
| [`skills/mobile-game-dev/SKILL.md`](skills/mobile-game-dev/SKILL.md) | **The rules.** Day-one decisions, reuse, design system, rendering performance, measuring on a real phone, architecture, testing, scope, and working habits for Claude Code |
| [`case-studies/`](case-studies/) | The three development logs the rules come from, with measurements, the bugs found and how |
| [`tools/device.mjs`](tools/device.mjs) | Attach to your game's WebView on a USB phone: run JS, measure frame rate during an action, list errors, take screenshots |

## Use it

**As a skill (recommended).** Claude loads it only when the task is about a mobile game.
```bash
# for all your projects
git clone https://github.com/Ahmad1765/claude-mobile-game-playbook
cp -r claude-mobile-game-playbook/skills/mobile-game-dev ~/.claude/skills/

# or for one project only
cp -r claude-mobile-game-playbook/skills/mobile-game-dev <your-project>/.claude/skills/
```

**Or as project instructions.** Paste the body of `SKILL.md` (everything below the `---` header) into your project's `CLAUDE.md`.

Then just work as usual ("start a new puzzle game", "the game lags when lines clear", "prepare the Play release"). You can also ask for it by name: "use the mobile-game-dev skill".

**The device tool:**
```bash
npm i -D playwright-core
APP_ID=com.example.game node tools/device.mjs fps "window.__dbg.confetti()" 3000
# {"syncMs":5.3,"fps":60,"late":"0%","worst":16.8,"frames":185}   (confetti, Helio P22)
```
Needs adb, USB debugging on the phone, and a debug build (Capacitor debug builds allow WebView debugging).

## The ten rules that mattered most
1. Check the game's name in the stores and fix the package id and signing key before writing code.
2. Copy your last project and trim it; never start from scratch.
3. Build the on-phone measurement tools before the features, and set a frame-rate budget on your weakest phone.
4. Make a design system (tokens, SVG icons, components) before screens. No emoji in the UI.
5. Redraw only on change; move things with compositor transforms; never read layout in a frame loop.
6. Test at 1.15× and 1.3× system text size, and with the real ad banner loaded.
7. Keep game rules pure and tested; validate generated and written content automatically.
8. Run an on-device regression after every change, with real touch input, failing on any page error.
9. Ship in small slices: game feel, then progression, then daily, then the rest.
10. Keep a DEVLOG per game, and read the last one before starting the next.

## Contributing
Found a rule that saved you, or one that's wrong for your setup? Open an issue or a pull request with what happened and, ideally, a measurement. Rules here should come from something that actually went wrong or right, not from opinion.

## License
[MIT](LICENSE)
