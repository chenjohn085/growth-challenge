# Growth Challenge (植物向性大挑戰) — Dev Notes

Classroom board game teaching plant tropism. Each group grows a virtual plant
(stem up into a sky-blue half, root down into a soil half) on a shared-shape
diamond-tiled board, over a fixed number of rounds. Highest total score wins.

**File:** `index.html` in this folder — single self-contained HTML/CSS/JS
file, no build step, no external dependencies (fonts, icons and sound are
all inlined/generated). Open it directly in a browser, publish it as a
Claude Artifact, or just visit the live GitHub Pages URL (this folder is
also a git repo pushed to `chenjohn085/growth-challenge`, served at
`https://chenjohn085.github.io/growth-challenge/`).

## Workflow for editing this file

1. Read the relevant section with `Read` (it's ~2200 lines; use `grep`/line
   offsets rather than reading it all at once).
2. Edit with `Edit`.
3. Before publishing, always run these checks (all zero-cost, all have
   caught real bugs during development):
   - `node -e "new Function(script)"` on the extracted `<script>` contents —
     catches syntax errors.
   - Cross-check every `el('id')` / `getElementById('id')` call against
     actual `id="..."` attributes in the HTML — catches typos/renames that
     silently `null`-crash at runtime.
   - Tag-balance count for `div`/`button`/`svg`/`span` — catches unclosed
     tags from a bad `Edit`.
4. Publish with the `Artifact` tool, same `file_path` each time, to keep the
   same URL.

No test suite exists — this is a single-file artifact, verified by the
static checks above plus manual play-testing in the browser.

## Screen flow

Each screen is a top-level `<div>` toggled via `show()`/`hide()` (just
`hidden` attribute). One `state` object (created in `el('startBtn')`'s
handler) drives everything from setup onward; `null` before that.

```
setupCard → readyScreen → tutorialScreen (interactive) → turnIntro
  → choiceOverlay (slot-machine draw reveal) → growthScreen (pick cells)
  → overviewScreen (between turns) → ... loops until maxRounds ...
  → resultsScreen
```

`resetToSetup()` (wired to the always-visible `.restart-fab`, with a custom
confirm modal since `window.confirm()` is blocked inside the Artifact
sandbox) can jump back to setup from anywhere.

## Data model

```js
state = {
  radius, groups, currentIndex, viewingIndex, round, maxRounds,
  pending,          // null, or the in-progress growth/setback step
  allowSetbacks,    // true = "Hard" mode, false = "Easy" mode (see Game modes)
  turnWallBonusUsed // reset to false at the start of every group's turn
}
```

Each entry in `groups`:

```js
{
  name, color,
  cells: {},        // "col,row" -> 'seed' | 'stem' | 'root'
  stemOrder: [], rootOrder: [],   // insertion order, used to find safe-to-remove leaves
  resources: {},     // "col,row" -> 'water' | 'nutrient' | 'sunlight', consumed on growth (+1 pt each)
  walls: {},         // "col,row" -> true, permanent
  litCells: {},      // "col,row" -> true, permanent "this stem cell caught sunlight" marker
  score: 0,          // accumulated resource points only
  finished: false,   // true once every direction is out of legal targets
  drawDeck: [], drawIndex: 0   // this group's own shuffled 'growth'/'damage' sequence
}
```

`totalScore(group) = (occupied cell count - 1) + group.score` — the `-1`
excludes the seed. Growing a cell is worth 1 point on its own; landing on a
resource is +1 more.

## Board geometry

Diamond lattice: a cell at `(col,row)` sits at
`left = cx + col*spacing, top = cy + row*spacing`, and is individually
`rotate(45deg)`'d via CSS. Only cells with `|col| <= |row|` are ever rendered
("bowtie" shape) — `row < 0` is the stem half (sky), `row > 0` is the root
half (soil), `row === 0` is just the seed. Radius `R` bounds
`max(|col|,|row|) <= R`.

Diagonal neighbor deltas (used everywhere: growth, wall adjacency, sunlight
shadow-casting, vine rendering):

```js
UL:{dc:-1,dr:-1}  UR:{dc:1,dr:-1}  DR:{dc:1,dr:1}  DL:{dc:-1,dr:1}
```

**Important, easy to re-derive wrong:** after a cell's own `rotate(45deg)`,
its SVG's native **top** edge faces screen **up-right**, **right** faces
**down-right**, **bottom** faces **down-left**, **left** faces **up-left**.
This is why a raised/embossed `box-shadow` on `.cell.wall` needs an *equal*
horizontal and vertical offset (e.g. `4px 4px`) to land as a straight-down
shadow on screen — the rotation turns a naive `0 4px` into a diagonal.

## Growth rules

```js
BRANCHES = {
  stem: { part:'stem', deltas: [all 4 diagonals] },
  root: { part:'root', deltas: [all 4 diagonals] }
}
```

Both branches grow in all 4 diagonal directions (no longer restricted to
"away from the seed" — that restriction was tried and explicitly reverted).
`computeGrowthTargets(group, outcome)` walks every existing seed/same-part
cell, and offers any *unoccupied* neighbor that's within radius, inside the
bowtie, on the correct half (`stem` blocked at `row>=0`, `root` blocked at
`row<=0` — explicit checks, not just relying on the seed being the only
crossing cell), and not a wall.

**Loop handling:** growth is *never* blocked for creating a structural loop
(two branches meeting up again). Instead, `buildSpanningParents()` runs one
BFS per render from the seed and `treeConnections()` only reports the edges
that BFS actually used — so a data-level loop still renders as a clean
branching tree. This was a deliberate reversal: an earlier version blocked
loop-forming moves outright, which made legal-looking cells suddenly
unclickable, and that was rejected.

## Resources & walls

All four token types (`water`, `nutrient`, `sunlight`, walls) use the same
target count: `tokenCountFor(R) = max(2, round(R * 0.9))`.

- **Walls** (`generateWalls`): placed one at a time from a shuffled pool,
  keeping a candidate only if `upperHalfFullyReachable()` (a BFS from the
  seed over non-wall cells) still holds — a **hard guarantee** that no wall
  layout can ever strand a cell, not just a probabilistic one.
- **Water/nutrient** (`generateResources`): straightforward fixed-count
  split from a shuffled pool of lower-half cells. Always exactly
  `tokenCountFor(R)` of each, deterministically, for every group.
- **Sunlight**: candidates are upper-half cells with a clear diagonal
  line-of-sight back to the board edge along `(-1,-1)` (light comes from the
  top-left at the same 45° as the grid) — a wall in the way blocks it.
  Because this depends on each group's own random wall layout, the number of
  *eligible* cells can differ group to group. **Fairness fix:** setup
  computes every group's wall layout and sunlight-candidate pool first
  (`sunlightCandidates`), takes `sunCount = min(tokenCountFor(R), every
  group's candidate-pool size)`, then calls `placeSunlight(candidates,
  sunCount)` per group — guaranteeing identical sunlight counts across
  groups even though walls/nutrient/water were already equal by construction.

## Draw / outcome system

```js
OUTCOMES = [stem, root, prune, rot, bonus_stem ("Strong Growth", 2 cells),
            bonus_root ("Deep Roots", 2 cells)]
```

**Fairness deck:** each group gets its own `buildDrawDeck(maxRounds,
allowSetbacks)` — a pre-shuffled array of exactly `maxRounds` entries
(`round(maxRounds * SETBACK_CHANCE)` of them `'damage'`, rest `'growth'`),
consumed in order via `drawIndex`. This replaced an earlier
`Math.random() < SETBACK_CHANCE` per-draw roll so that every group
experiences the *same number* of setbacks over a game, just shuffled
differently — no group can get unlucky with a streak of bad draws. When
`allowSetbacks` is false the deck is just 100% growth.

**Wall bonus:** growing a stem cell diagonally adjacent to a wall
(`isAdjacentToWall`), once per turn (`state.turnWallBonusUsed`), triggers
`startWallBonus()` — offers **1** extra free stem cell from the *current
full stem growth pool* (not just neighbors of the trigger cell). The
resulting message combines points from the triggering step and the bonus
into one `+N points` total rather than showing two separate lines.

**No-effect draws:** if a growth draw's whole branch has zero legal targets,
or a setback draw's part has zero safely-removable cells
(`removableCells` — BFS connectivity check so a setback can never split the
plant, only trim a leaf), `enterNoEffectPhase()` shows the card with a red
"No Effect" label and a denial sound instead of silently skipping.

## Rendering

- **Connected-vine autotile:** a grown cell doesn't fill its diamond flat;
  `connectionShapeSvg()` draws a rounded "arm" toward each neighbor that's
  the same part (via `treeConnections`, see Loop handling above), plus a hub
  circle when 2+ arms meet — so a run of cells reads as one continuous vine.
- **`renderBoardInto(stageEl, layerEl, group, interactive, simplified, bloomKey)`**
  is the one render function used everywhere (main board, draw-reveal board,
  overview thumbnails, results thumbnails, tutorial board).
  `simplified=true` skips grid/resource/wall rendering entirely and skips
  ungrown cells — used for every "just show the plant" context (mini boards,
  tutorial's final screen). `bloomKey` (from `topStemCellKey(group)`, the
  stem cell with the smallest/most-negative row) draws a flower overlay
  there — used for the winner in the results screen and for the tutorial's
  final "You're Ready!" screen.
- **Overview/results grid layout:** deliberately `display:flex;
  flex-wrap:wrap; justify-content:center` with the *container* capped at
  `max-width: 586px` (exactly 2×280px mini-stage + 1×26px gap), rather than
  giving each item a percentage `flex-basis` — the mini-stage has a fixed
  280px width, and fighting that with a computed basis caused visible
  overlap. This also naturally centers an incomplete last row, which CSS
  Grid does not do without extra work.

## Sound effects

All procedural Web Audio (`playChime(freqs, noteDur, type)`), no audio
files: `sfxTick`/`sfxLand` (slot machine), `sfxScoreUp` (called at the exact
mutation point in `onCellClick`, not tied to the score-counter animation —
tying it to "did the displayed number change" false-triggered when a reused
chip element just switched to showing a *different* group's already-existing
score), `sfxSetback` (descending triangle motif, for a setback actually
landing — distinct from `sfxDenied`'s harsh sawtooth, which is for a
blocked/no-effect draw), `sfxResults`.

## Game modes

Setup screen has an Easy/Hard toggle (`#modeToggle`, first field in the
setup card). Stored as `allowSetbacks` on `state`. Hard = setbacks can be
drawn (25% of a group's deck). Easy = deck is 100% growth, and the
interactive tutorial skips its Pruned!/Root Rot! steps accordingly
(`buildTutorialSteps()` reads `state.allowSetbacks`).

## Interactive tutorial

`#tutorialScreen`, driven by a small **separate, scripted** engine
(`buildTutorialGroup`, `buildTutorialSteps`, `startTutorial`,
`showTutorialStep`, `handleTutorialTap`) — deliberately not wired through
the real game's `state.pending`/draw machinery, since every tutorial step is
a fixed, hand-picked cell rather than a random draw, and it must never touch
real `state.groups`.

- `buildTutorialGroup()` hand-places sunlight/water/nutrient/a wall at
  specific coordinates (all within radius 3, the minimum board size, and
  verified not to be wall-adjacent before the dedicated "Wall" step — an
  earlier layout accidentally put the "Sunlight" step's target cell
  diagonally against the wall, which would have silently qualified for a
  wall bonus before wall bonuses were even explained).
- 9–11 steps (2 fewer in Easy mode): Grow Stem → Strong Growth → Grow Roots
  → Deep Roots → Sunlight → Water → Nutrients → Wall → Wall Bonus →
  [Pruned! → Root Rot!, Hard mode only]. Sunlight/Water/Wall are each their
  own step with their own card — deliberately *not* riding along on a
  "grow stem/root" tap, so each concept gets its own explanation.
  A `Skip Tutorial` ghost-button is always visible and jumps straight to
  `turnIntro`.
- Has its own score chip (`#tutorialScoreNum`), using the same
  `totalScore()` formula as the real game, so it accumulates from the very
  first tap exactly like the real score does.
- Final "You're Ready!" screen renders the practice plant in `simplified`
  mode with a bloom at `topStemCellKey()`, then shows the win-condition text
  and the button into `turnIntro`.

## Notable rejected/reverted approaches (don't redo these)

- Blocking growth moves that would form a structural loop — rejected, made
  legal-looking cells unclickable. Loops are hidden at render time instead
  (see Loop handling above).
- `window.confirm()` for the restart confirmation — silently does nothing;
  the Artifact iframe sandbox blocks native dialogs. Use the custom
  `#confirmOverlay` modal pattern instead for anything needing confirmation.
- CSS Grid (`repeat(4,1fr)`) for the thumbnail grids — doesn't center an
  incomplete last row. Flex + capped container width does.
- Tying `sfxScoreUp()` to the score-counter animation helper — false-fires
  when a reused DOM element just switches to a different group's existing
  score. Sound is called explicitly at the actual mutation point instead.
