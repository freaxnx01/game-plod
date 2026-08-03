# Guided tutorial level — design

Issue: [freaxnx01/game-plod#4](https://github.com/freaxnx01/game-plod/issues/4)

## Problem

New/younger players have no onboarding — the ready screen shows a one-line
instruction and hotkey hints, then drops the player straight into Moss
Hollow with all 8 skills available and a timer running. There's no guided,
learn-by-doing path to understanding what each skill does before being
expected to use it under time pressure.

## Scope

The original issue bundled two independent features: a hint system (in-game
help available anytime) and a tutorial (onboarding). These were split:
this spec covers **only the tutorial**. The hint system is tracked
separately in [freaxnx01/game-plod#6](https://github.com/freaxnx01/game-plod/issues/6),
parked for now.

- A new, purpose-built tutorial level teaching all 8 skills in a fixed
  linear sequence, reached via a new `TUTORIAL` button on the ready screen
  (optional — normal `PLAY` is unaffected).
- Untimed, no lose condition — matches the "for younger players" framing:
  no time pressure while learning.
- 3-5 plods, same OUT/HOME spawn/rescue mechanic as normal play, just
  without the timer/quota-fail mechanics.
- Out of scope: the hint system (#6), any change to Moss Hollow itself,
  `identity.html`, i18n of the new tutorial text (issue #2's i18n
  mechanism is a separate, unsequenced issue — if it lands first, new
  tutorial strings should ideally flow through it, but this spec does not
  depend on or require it; if #2 hasn't landed, tutorial strings are added
  as plain English literals like the rest of the pre-i18n codebase).

## Design

### Multi-level plumbing

`Level` currently hard-codes one level (Moss Hollow: `W`/`H`/`HATCH`/`EXIT`/
`steel`/terrain shapes, all as class fields, confirmed via investigation —
terrain is procedurally drawn Canvas 2D paths rasterized into a grid at
init time, not an image asset or embedded data blob). A second level
definition, `TUTORIAL_LEVEL`, is added with its own terrain shapes,
`HATCH`/`EXIT`/`W`/`H`, `steel` list, and a `plodCount` of 3-5 (vs. Moss
Hollow's normal `totalPlods` prop). `Level.initLevel()` takes a `levelId`
parameter (`"moss-hollow"` default, `"tutorial"` when launched via the new
button) and branches to build the right terrain/constants. The tutorial
level omits `TIME`/quota-based lose logic entirely — reaching EXIT with
any plods home is treated as complete, there is no "too few made it" path
for this level.

### Obstacle sequence

One obstacle per skill, in this fixed left-to-right order (chosen for
increasing conceptual difficulty, not alphabetical/hotkey order):

1. **BUILDER** — a narrow gap that can only be crossed by bridging it.
2. **CLIMBER** — a wall taller than the walk-up threshold.
3. **BASHER** — a thin wall blocking the horizontal path at floor height.
4. **MINER** — a sloped mound requiring a diagonal descent.
5. **DIGGER** — a floor section that must be dug through to reach a lower
   area.
6. **FLOATER** — a high ledge the plod must drop from safely (floater must
   be assigned before the drop).
7. **BLOCKER** — a fork or hazard where a stray plod would wander off;
   placing a blocker redirects the group correctly.
8. **BOMBER** — a final wide obstruction only a bomber's crater can clear,
   opening the path to EXIT.

Each obstacle is placed so the *previous* obstacle's clearing is a
prerequisite to reach it (a straight, unbranching critical path) — a
player cannot stumble into obstacle 5 without having solved 1-4 first,
which is what makes strict step-locking (below) coherent.

### Prompt overlay and skill-locking

A banner anchored below the HUD strip (not overlapping the play area)
shows the current step's instruction in a consistent format: what to do,
naming the skill and the action (e.g. "Click BUILDER, then click a plod to
bridge the gap"). `state.tutorialStep` (0-7, or `null` when not in a
tutorial run) tracks progress. While a tutorial is active:

- Skill tiles other than the current step's named skill are visually
  dimmed (reduced opacity, matching the existing `dim` styling already
  used for skill tiles with zero remaining count) and their assignment is
  blocked — `assign()` gets an added check: if `state.tutorialStep != null`
  and the requested skill isn't `TUTORIAL_SEQUENCE[state.tutorialStep]`,
  reject the assignment (same no-op pattern as the existing
  zero-count-remaining rejection).
- Each step has a completion condition checked once per tick alongside the
  main game loop (not a separate polling system) — the specific state
  transition that skill's use produces (e.g. step 1 completes when a plod
  that was in `"builder"` state transitions back to `"walker"` past the
  gap's x-coordinate; step 6 completes when a floater-flagged plod lands
  from a fall past the ledge's x-coordinate). When a step's condition is
  met, `state.tutorialStep` increments and the banner updates to the next
  instruction.
- After step 8 completes and any plod reaches EXIT, a tutorial-complete
  overlay (distinct from the normal win screen — no stats, just a
  congratulatory message and a button back to the main ready screen)
  is shown.

### Known risk: blind geometry authoring

Terrain is hand-authored Canvas path code (confirmed: `paintTerrain()`
draws shapes via `lineTo`/`quadraticCurveTo`/`fill`, then `buildGrid()`
rasterizes via `getImageData` alpha-thresholding — there is no image asset
or embedded per-pixel data to copy/adapt). An implementing agent designing
8 new obstacles' worth of coordinates is authoring geometry without being
able to "see" the result while writing code. This is a real risk of
producing terrain that doesn't render sensibly or isn't actually solvable
with the intended skill. **Mitigation, required in the implementation
plan:** after authoring each obstacle's terrain, render the level in a
running browser and take a screenshot to visually confirm the shape looks
correct and is solvable, before moving to the next obstacle — do not
author all 8 blind and check at the end.

## Testing / verification

No automated test suite exists in this repo (confirmed for issues #5, #2,
#3 — single self-contained `index.html`, no test framework). Verification
is manual playtest plus the mandatory visual/screenshot verification above:

1. Launch the tutorial from the new `TUTORIAL` button. Confirm the banner
   shows step 1's instruction and only BUILDER is enabled (other 7 tiles
   dimmed/unclickable).
2. Solve each of the 8 obstacles in order; confirm the banner and unlocked
   skill advance correctly after each, and that no earlier/later skill is
   assignable out of turn.
3. Confirm no timer is shown and no lose condition can trigger regardless
   of how long the level takes.
4. Reach EXIT after step 8 — confirm the tutorial-complete overlay appears
   (not the normal win screen) and its button returns to the main ready
   screen with normal `PLAY`/`TUTORIAL` options intact.
5. Confirm Moss Hollow (`PLAY`) is completely unaffected — same terrain,
   timer, quota, and skill counts as before this change.

## Out of scope

- The hint system (issue #6).
- Any change to Moss Hollow's terrain, timer, or quota.
- `identity.html`.
- i18n of new tutorial strings (independent of, not blocked by, issue #2).
- A skip/replay-tutorial-anytime affordance beyond the one `TUTORIAL`
  button on the ready screen — not requested, not designed here.
