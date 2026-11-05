# CLAUDE.md

Personal GitHub Pages site (served at https://cloudallen.github.io) for David Jose de Melo Junior, a math Ph.D. student at IMPA: an academic landing page plus small browser puzzle games built as side projects.

## Stack and deployment

- Plain static HTML. Each page is a single self-contained file with inline `<style>` and `<script>`. No build step, no package manager, no framework, no tests.
- GitHub Pages serves the `main` branch root directly, so anything pushed to `main` goes live.
- To preview locally, open the file in a browser or run `python -m http.server` from the repo root.

## Files

| File | Purpose |
|---|---|
| `index.html` | Landing page / academic resume. Links to the side projects using absolute `https://cloudallen.github.io/...` URLs. |
| `PegSolitaire.html` | Interactive 15-hole triangular peg solitaire on a `<canvas>` (400x420), with "Solve Me" and "Anti-Solve" buttons. |
| `slidingpuzzle.html` | Rush Hour–style 6x6 sliding car puzzle (DOM-based, pointer events). It has 30 puzzles from Michael Fogleman's database in 5 difficulty tiers, plus the original "The Classic" level as a 6th tab. It also has undo, hints, an auto-solver and a box for pasting any database board string. |
| `PegSolitaire-Solutions.html` | Large (~330 KB) Jupyter notebook exported with nbconvert (`RestaUm`). Lists precomputed solution / anti-solution move sequences as ASCII boards. Generated output: don't hand-edit it, regenerate it from the notebook (the notebook isn't in this repo). |
| `gamenight.html` | Bilingual (PT/EN) promo page for the weekly IMPA game night (Thursdays 18:30–21:00): info cards, game catalog with rules summaries. |
| `mimica.html` | Bilingual (PT/EN) word generator for mime and drawing games: draws random words from chosen categories in English, Portuguese or both. |
| `mimic_words_EN_PT.txt` | Source word list for `mimica.html` (33 categories, 3,363 `English \| Português` lines). The page doesn't read this file at runtime. |
| `README.md` | Short description with links to each page. |

## Game internals

### Peg solitaire (`PegSolitaire.html`)
- The board has 15 holes, indexed 0–14 row by row from the top (row `r` has `r+1` holes). Index labels are drawn under each hole.
- `MOVES[s]` lists `[over, end]` pairs: a peg at `s` can jump over `over` into `end`. This table is the single source of truth for legal moves, used by both the solver and the drag-and-drop handling.
- `pegs` is a 15-element boolean array. `resetGame(idx)` empties one starting hole. The buttons only offer holes 0, 1, 3 and 4, which cover every case up to symmetry.
- `solve(state, findBest)` is an exhaustive recursive DFS. `findBest=true` minimizes the remaining pegs (it stops early at 1). `false` maximizes them (the "anti-solution": the most gridlocked dead end, which is the author's research interest). `runSolver` animates the path at 600 ms per move.
- Touch input works by re-dispatching touch events as synthetic `MouseEvent`s.

### Rush Hour (`slidingpuzzle.html`)
- Puzzles come from https://www.michaelfogleman.com/rush/ (cited in the page footer). `TIERS` holds `[boardString, optimalMoves]` pairs. A board string has 36 characters, row-major: `o` empty, `x` wall, `A` red car (always horizontal on row 2), `B`–`Z` other vehicles. One move slides one vehicle any distance. If you add puzzles, verify the move counts with `solveFrom`.
- `parseBoard` turns a string into `pieces` (`{id, horiz, len, fixed, pos}`, with `pieces[0]` = red car) and `walls`. The game state is just the array of `pos` values. `occupancy` and `slideRange` are shared by dragging and the solver.
- Everything on the board is positioned with `calc(var(--cell) * n)`. `updateCellSize` sets `--cell` (40–60px) from the window width so the board fits on phones.
- Dragging uses Pointer Events, with `touch-action: none` on the board and the vehicles so the page doesn't scroll on mobile.
- `solveFrom` is an uncapped BFS (database clusters are small, and the hardest puzzles solve in tens of ms). The win condition is the red car's `pos === 4`, after which it animates off the board.
- The URL hash holds the current board string, so links like `slidingpuzzle.html#<board>` load any puzzle. Puzzles the player solved without the auto-solver are stored in `localStorage` (`rushHourSolved`) and shown with a check mark.

### Game Night (`gamenight.html`)
- All editable content is at the top of the `<script>`: `CONFIG` (the room name) and `GAMES` (catalog plus `rules.{pt,en}.{objective,setup,turn,end}`, optional `link`). The game list is the real collection. Saboteur, Uno, Terra à Vista (Toyster), Imagem & Ação (Grow's Pictionary, which the user calls "Pictionaire"), Werewolf and Spyfall have rules summaries. There are four homemade games (`category: 'custom'`). Crack and Owl's Ass have rules, written from the user's own notes. Avalon has only a description ("card-game version of The Resistance"). Avalon and Caracaca still show "Rules coming soon". Crack's `minutes: '∞'` is an intentional joke.
- Card pictures (`gamePicture`): `image` (a photo path) if set, otherwise the inline SVG symbol `art-<id>` from the sprite at the top of `<body>`, otherwise the name's first letter. New games need a new `art-<id>` symbol (64x64 viewBox, `currentColor` fill, `#07170f` details) or they fall back to the letter.
- Only `id` and `name` are required. `players: [min]` (no max) displays as "min+". If set, `category` must be a key of `STRINGS[lang].cat` and of `CAT_COLORS`.
- i18n: static text uses `data-i18n` keys from `STRINGS`. `setLang` re-renders everything and stores the choice in `localStorage` (`gamenight-lang`). The default comes from `navigator.language`.

### Mime & Draw (`mimica.html`)
- The word list is an exact copy of `mimic_words_EN_PT.txt`, embedded in `<script type="text/plain" id="words">` so the page also works from `file://`. If the txt file changes, paste it into that block. `parseWords` reads it at startup: a line between two `====` lines is a category header, any other line with `|` is a word.
- Editable content is at the top of the main `<script>`: `PRESETS` (category lists use the English names from the word list), `CATEGORY_ICONS` (keyed the same way; a new category falls back to `DEFAULT_ICON`) and the `COUNT_*` limits for words per draw.
- `buildPool` drops pairs that appear in more than one category (3,363 lines give 3,316 distinct words), so one draw never repeats a word. Words can repeat between draws.
- Timer: when `settings.timerOn` is set, `draw` calls `playDrawSound` and `startTimer`. A sticky panel counts down from `settings.timerSeconds`, turns red for the last `TIMER_WARNING` seconds, and `tick` plays `playTimeUpSound` at zero. Drawing again restarts it, and the Stop button cancels it without the alarm. Sounds are synthesized with Web Audio (`tone`), so there are no audio files; the `AudioContext` has to be created on the Draw click because browsers block sound until a user action. A Screen Wake Lock keeps phones awake during a run.
- Settings (preset, custom categories, count, word language, timer) are stored in `localStorage` (`mimica-settings`) and validated in `loadSettings`. The interface language shares the `gamenight-lang` key with `gamenight.html`.
- `syncSettings` updates everything that depends on settings without rebuilding controls, so checkboxes keep focus. `renderSettings` rebuilds the chips and checklist, and only runs on a language change.

## Conventions

- Keep each page self-contained: inline CSS/JS, no external dependencies (the nbconvert export is the exception).
- Mobile support matters, since several commits fix touch handling. Test drag interactions on touch as well as mouse.
- When you add a project, add a `.section-item` under "Side Projects" in `index.html` and a link in `README.md`. Use the same absolute-URL link style, or a relative link.
