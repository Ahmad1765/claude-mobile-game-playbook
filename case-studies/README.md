# Case studies

The development logs the rules in `../skills/mobile-game-dev/SKILL.md` were drawn from, lightly edited to remove personal details. They are kept as written, so file paths and links point into the original game projects, which are not part of this repo.

| Log | Game | What it shows |
|---|---|---|
| [1-block-puzzle-gemstack.md](1-block-puzzle-gemstack.md) | Block puzzle (first game) | Four rewrites: lag from canvas redraws, effects frame drops, the full retrospective (§5) |
| [2-mahjong-lotus.md](2-mahjong-lotus.md) | Mahjong pairs | Applying the retrospective from day one; edge-to-edge on Android 10; storage flush quirk |
| [3-word-search-word-passport.md](3-word-search-word-passport.md) | Word search | Reuse-first build, content validation, large-text bugs, process traps |
| [4-arrow-puzzle-arrow-garden.md](4-arrow-puzzle-arrow-garden.md) | Arrow tap-away puzzle | Solvable-by-construction generator, name collisions, confounded fps measurement |
| [5-color-shooter-mosaic-parade.md](5-color-shooter-mosaic-parade.md) | Conveyor colour shooter | A shared simulation for game, solver and tests; balancing with a random player; DOM tokens at 60 fps |
| [6-goods-sort-shopkeeper-trio.md](6-goods-sort-shopkeeper-trio.md) | Goods triple-sort | Proving the never-stuck rule; a DOM board patched per cell; SVG images vs PNG on a weak GPU |
| [7-word-solitaire-word-parlour.md](7-word-solitaire-word-parlour.md) | Word association solitaire | The owner's "too similar, too AI-made" verdict and the fix: a named style per game, outside CC0 resources, an approved selling point; content rules; em vs text zoom |
| [8-sudoku-quiet-sudoku.md](8-sudoku-quiet-sudoku.md) | Sudoku and killer sudoku | Testing outside libraries before adopting them; a human-technique solver for grading, uniqueness and explaining hints; a pre-built puzzle pack; CC0 museum art |
| [9-block-jam-bauhaus-blocks.md](9-block-jam-bauhaus-blocks.md) | Sliding block jam | A* with an admissible bound for provably shortest solutions; rejecting impossible pieces at generation; a parallel deterministic level build |
| [10-bus-sort-last-bus-home.md](10-bus-sort-last-bus-home.md) | Passenger colour sort | A fresh project per game (no folder copying); the game state as a set; animation queues tied to their game; CC0 open data cleaned by script |
| [11-triple-match-tile-picnic.md](11-triple-match-tile-picnic.md) | Triple tile match | Boards solvable by construction (the right invariant); finding a GPU-raster frame problem with a trace; a decent-player difficulty measure |
| [12-tripeaks-peak-lodge.md](12-tripeaks-peak-lodge.md) | TriPeaks solitaire | Facts from Wikidata instead of memory; local calendar dates; build-time rendering of heavy public-domain art; checking listing claims against code |
| [13-screw-puzzle-clockwork-bolts.md](13-screw-puzzle-clockwork-bolts.md) | Screw / bolt puzzle | Closed-form "physics" instead of an engine; knife-edge rejection with rounding slack; ImageBitmap sprites; a DevTools first-frame measurement artifact |
| [14-ludo-courtyard.md](14-ludo-courtyard.md) | Ludo (board game) | A host-as-referee online mode on Supabase Realtime; validating every message from other phones; checkable fair dice; a negative animation time that froze the game |
| [15-hole-game-gobble-town.md](15-hole-game-gobble-town.md) | Hole / swallow arcade | Real towns from OpenStreetMap; a tested size ladder; two-layer canvas tiles; a duplicate animation loop that skewed every measurement |
| [16-idle-restaurant-bun-street.md](16-idle-restaurant-bun-street.md) | Idle arcade (carry stacks) | An economy simulated before visuals, which found two stuck states; canvas at CSS pixels; device-test inputs that fail silently |
