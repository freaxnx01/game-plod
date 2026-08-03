# Guided Tutorial Level Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an optional, untimed, no-fail tutorial level to `index.html`
that teaches all 8 skills in a fixed sequence via auto-advancing prompts
and per-step skill-locking, reached via a new `TUTORIAL` button.

**Architecture:** `Component` (the single class holding all game state and
logic — confirmed there is no separate `Level` class) gets a
`TUTORIAL_LEVEL` config class field alongside the existing `SKILLS`/`W`/`H`
fields, and a `TUTORIAL_SEQUENCE` array (8 entries: `{ skill, gateX,
prompt }`). `initLevel()` branches on a new `state.levelId` (`"moss"` |
`"tutorial"`) to override `this.W`/`H`/`HATCH`/`EXIT`/`steel`/`kill` and
call a new `paintTutorialTerrain()` instead of `paintTerrain()`. Step
completion is detected the simplest possible way: **a step is complete the
moment any plod's x-position passes that step's `gateX`** — since the gate
sits physically past the obstacle, a plod cannot reach it without having
cleared that obstacle by definition, so this needs no per-skill state-
transition detection. Skill-locking (only the current step's skill is
assignable) — not terrain material — is what enforces "use skill X here";
obstacles don't need to be mechanically unsolvable by other skills, since
those are UI-disabled during each step.

**Tech Stack:** Vanilla JS, single self-contained `index.html`, no build
step, no test framework, Canvas 2D for terrain.

## Global Constraints

- `index.html` is normally a generated dc-tool bundle; no source file
  exists in this repo for game-plod — hand-editing `index.html` directly
  is the approved approach (confirmed for issues #5, #2, #3).
- Everything lives inside one `data-dc-script=""` JSON-escaped string:
  literal newlines are `\n` (backslash+n), quotes are `\"`
  (backslash+quote). Copy every snippet in this plan verbatim, including
  that escaping.
- This file is in its **pre-i18n state** (issue #2's `STRINGS`/`t()`
  mechanism is a separate, unsequenced issue, not assumed to exist). New
  tutorial strings are added as plain literals, matching the rest of the
  current codebase.
- No `Level` class exists — `W`, `H`, `HATCH`, `EXIT`, `QUOTA_FRAC`,
  `TIME`, `MAX_FALL`, `steel`, `kill`, `SKILLS` are all fields/locals on
  the single `Component extends DCLogic` class. New tutorial config goes
  in the same place.
- **Mandatory visual verification:** terrain is 100% procedural Canvas
  path code with no image asset to reference — an implementer authoring
  new shapes cannot "see" the result while writing code. After adding
  *each* obstacle's terrain in Task 2, render the level in a browser
  (`python3 -m http.server 8123` from the repo root, open
  `http://localhost:8123/index.html`, and — since there is no automated
  screenshot tooling in this repo — use whatever interactive browser/
  screenshot capability is available in your environment to look at the
  actual rendered canvas) and confirm the shape looks correct and is
  physically walkable/solvable before moving to the next obstacle. Do not
  author all 8 obstacles and check at the end.
- No automated test suite exists in this repo. All other verification is
  manual playtest.
- Surgical edits only — don't touch Moss Hollow's terrain, timer, or
  quota; don't touch `identity.html`.

---

### Task 1: Multi-level plumbing — TUTORIAL_LEVEL config, levelId branching, TUTORIAL button

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: `TUTORIAL_LEVEL` (class field: `{ W, H, HATCH, EXIT, steel,
  kill, plodCount }`), `state.levelId` (`"moss"` default | `"tutorial"`),
  `initLevel()` branching on it, `total()`/`quota()` branching on it, a
  `startTutorial` click handler, a `TUTORIAL` button on the ready screen.
- Consumes: nothing from other tasks (foundation task).

- [ ] **Step 1: Add the TUTORIAL_LEVEL config field**

Find this exact literal text in `index.html` (the existing level-constant
class fields, right after `SKILLS`):

```
W = 1360; H = 727; QUOTA_FRAC = 0.5; TIME = 360; MAX_FALL = 250;\n  HATCH = { x: 180, y: 216 }; EXIT = { x: 1180, y: 470 };\n
```

Replace it with:

```
W = 1360; H = 727; QUOTA_FRAC = 0.5; TIME = 360; MAX_FALL = 250;\n  HATCH = { x: 180, y: 216 }; EXIT = { x: 1180, y: 470 };\n  TUTORIAL_LEVEL = {\n    W: 2000, H: 780,\n    HATCH: { x: 60, y: 150 }, EXIT: { x: 1950, y: 700 },\n    steel: [],\n    kill: [],\n    plodCount: 4\n  };\n  TUTORIAL_SEQUENCE = [\n    { skill: \"builder\", gateX: 300, prompt: \"Click BUILDER, then click a plod \\u2014 bridge the gap.\" },\n    { skill: \"climber\", gateX: 500, prompt: \"Click CLIMBER, then click a plod \\u2014 scale the wall.\" },\n    { skill: \"basher\", gateX: 700, prompt: \"Click BASHER, then click a plod \\u2014 bash through.\" },\n    { skill: \"miner\", gateX: 990, prompt: \"Click MINER, then click a plod \\u2014 dig down the slope.\" },\n    { skill: \"digger\", gateX: 1180, prompt: \"Click DIGGER, then click a plod \\u2014 drop through the floor.\" },\n    { skill: \"floater\", gateX: 1450, prompt: \"Click FLOATER, then click a plod \\u2014 float off the ledge.\" },\n    { skill: \"blocker\", gateX: 1750, prompt: \"Click BLOCKER, then click a plod \\u2014 turn the group away from the pit.\" },\n    { skill: \"bomber\", gateX: 1900, prompt: \"Click BOMBER, then click a plod \\u2014 blast through to the exit.\" }\n  ];\n
```

Note: `TUTORIAL_LEVEL.steel`/`kill` are empty arrays for now — the level
as designed doesn't need indestructible-steel or foil-hazard zones (the
floater obstacle's fall distance, not a hazard zone, is what makes that
lesson meaningful — see Task 2 Step 6). `gateX` values are starting
points matching the terrain layout authored in Task 2 — adjust them if
Task 2's screenshot verification reveals an obstacle's actual cleared
position differs from these estimates.

- [ ] **Step 2: Branch initLevel() on levelId**

Find this exact literal text (the start of `initLevel()`):

```
initLevel() {\n    const W = this.W, H = this.H;\n    this.bg = document.createElement(\"canvas\"); this.bg.width = W; this.bg.height = H;\n    this.terrain = document.createElement(\"canvas\"); this.terrain.width = W; this.terrain.height = H;\n    this.tc = this.terrain.getContext(\"2d\");\n    this.steel = [{ x: 850, y: 618, w: 130, h: 82 }];\n    this.kill = [{ x: 648, y: 658, w: 164, h: 62 }];\n    this.paintBG(); this.paintTerrain(); this.buildGrid();\n
```

Replace it with:

```
initLevel() {\n    const isTutorial = this.state.levelId === \"tutorial\";\n    if (isTutorial) {\n      const tl = this.TUTORIAL_LEVEL;\n      this.W = tl.W; this.H = tl.H; this.HATCH = tl.HATCH; this.EXIT = tl.EXIT;\n    } else {\n      this.W = 1360; this.H = 727; this.HATCH = { x: 180, y: 216 }; this.EXIT = { x: 1180, y: 470 };\n    }\n    const W = this.W, H = this.H;\n    this.bg = document.createElement(\"canvas\"); this.bg.width = W; this.bg.height = H;\n    this.terrain = document.createElement(\"canvas\"); this.terrain.width = W; this.terrain.height = H;\n    this.tc = this.terrain.getContext(\"2d\");\n    this.steel = isTutorial ? this.TUTORIAL_LEVEL.steel : [{ x: 850, y: 618, w: 130, h: 82 }];\n    this.kill = isTutorial ? this.TUTORIAL_LEVEL.kill : [{ x: 648, y: 658, w: 164, h: 62 }];\n    this.paintBG(); if (isTutorial) this.paintTutorialTerrain(); else this.paintTerrain(); this.buildGrid();\n    this.tutorialStep = isTutorial ? 0 : null;\n
```

`this.paintTutorialTerrain()` is added in Task 2 — `initLevel()` will
reference a not-yet-defined method until then; that's expected, this task
isn't independently runnable until Task 2 lands (Task 2's own
verification step is what first exercises this branch).

- [ ] **Step 3: Branch total()/quota() for the tutorial's fixed plod count**

Find this exact literal text (the `total()`/`quota()` methods):

```
total() { return Math.round(this.props.totalPlods ?? 20); }\n  quota() { return Math.max(1, Math.round(this.total() * this.QUOTA_FRAC)); }\n
```

Replace it with:

```
total() { return this.state.levelId === \"tutorial\" ? this.TUTORIAL_LEVEL.plodCount : Math.round(this.props.totalPlods ?? 20); }\n  quota() { return this.state.levelId === \"tutorial\" ? this.TUTORIAL_LEVEL.plodCount : Math.max(1, Math.round(this.total() * this.QUOTA_FRAC)); }\n
```

The tutorial's quota equals its full plod count (all must get home) —
this only matters if the normal `finish()`/win-screen path were reached,
which Task 5 replaces for the tutorial with its own completion path, but
keeping `quota()` correct here avoids a dangling inconsistency in the
meantime.

- [ ] **Step 4: Add the TUTORIAL button and levelId-setting handlers**

Find this exact literal text (the PLAY button and the `startGame` handler
shown together — first the markup):

```
<div style=\"width:250px;height:66px;background:#FFFBF2;border:3px solid #2E7CD6;border-radius:14px;box-shadow:0 6px 0 rgba(43,35,32,.24);display:grid;place-items:center;font:600 23px Fredoka,sans-serif;color:#2B2320;cursor:pointer;animation:kpulse 1.6s ease-in-out infinite\" sc-camel-on-click=\"{{ startGame }}\">PLAY<\u002Fdiv>\n  <\u002Fsc-if>\n
```

Replace it with:

```
<div style=\"display:flex;gap:12px\">\n      <div style=\"width:250px;height:66px;background:#FFFBF2;border:3px solid #2E7CD6;border-radius:14px;box-shadow:0 6px 0 rgba(43,35,32,.24);display:grid;place-items:center;font:600 23px Fredoka,sans-serif;color:#2B2320;cursor:pointer;animation:kpulse 1.6s ease-in-out infinite\" sc-camel-on-click=\"{{ startGame }}\">PLAY<\u002Fdiv>\n      <div style=\"width:160px;height:66px;background:#FFFBF2;border:2.5px solid #2B2320;border-radius:14px;box-shadow:0 5px 0 rgba(43,35,32,.22);display:grid;place-items:center;font:600 17px Fredoka,sans-serif;color:#2B2320;cursor:pointer\" sc-camel-on-click=\"{{ startTutorial }}\">TUTORIAL<\u002Fdiv>\n    <\u002Fdiv>\n  <\u002Fsc-if>\n
```

Then find this exact literal text (the `startGame`/`restart` handlers):

```
startGame: () => this.setState({ phase: \"play\" }),\n      restart: () => this.doRestart(),\n
```

Replace it with:

```
startGame: () => this.setState({ phase: \"play\", levelId: \"moss\" }, () => this.initLevel()),\n      startTutorial: () => this.setState({ phase: \"play\", levelId: \"tutorial\" }, () => this.initLevel()),\n      restart: () => this.doRestart(),\n
```

Both handlers set `levelId` and immediately re-run `initLevel()` in the
`setState` callback (so the correct level's grid/HATCH/EXIT are built
before the first `play`-phase tick reads them) — this mirrors how
`doRestart()` already calls `initLevel()` directly, just gated by which
button was clicked. `doRestart()`/`R` reuses whatever `levelId` is
already set (restarting the tutorial restarts the tutorial, restarting
Moss Hollow restarts Moss Hollow) — no change needed there.

- [ ] **Step 5: Manual verification**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
python3 -m http.server 8123
```

Open `http://localhost:8123/index.html`. Confirm PLAY and a new TUTORIAL
button both appear on the ready screen, side by side. Clicking TUTORIAL
at this point in the plan will throw a JS error in the console
(`this.paintTutorialTerrain is not a function`) — that's expected until
Task 2 lands; confirm clicking PLAY still works exactly as before (Moss
Hollow loads and plays normally) to prove this task didn't regress the
existing flow.

- [ ] **Step 6: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add index.html
git commit -m "feat(tutorial): add TUTORIAL_LEVEL config, levelId branching, TUTORIAL button (#4)"
```

---

### Task 2: Tutorial terrain — paintTutorialTerrain(), 8 obstacles, screenshot-verified one at a time

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `TUTORIAL_LEVEL.W`/`H` from Task 1.
- Produces: `paintTutorialTerrain()` method (referenced by Task 1's
  `initLevel()` branch, called the same way `paintTerrain()` is for Moss
  Hollow).

**Terrain layout being authored** (a two-corridor stepped floor, straight
lines only — no curves, chosen specifically to keep the geometry easy to
reason about without rendering, unlike Moss Hollow's organic sawtooth
shapes):

- Upper corridor floor at `y=300`: `x0-200` start area, `x200-280` GAP
  (BUILDER bridges it), `x280-420` floor, `x420-460` a wall rising to
  `y=140` (CLIMBER), `x460-600` floor at `y=140`, `x600-640` a pillar
  rising further to `y=100` (BASHER tunnels through it), `x640-780` floor
  at `y=140`, `x780-950` a diagonal slope back down to `y=300`
  (MINER), `x950-1100` floor at `y=300`.
- `x1100-1160`: a thin floating slab (`y=300` to `y=340` only, open air
  both above and below) over a gap down to a lower corridor at `y=440`
  (DIGGER digs through the slab).
- Lower corridor floor at `y=440`: `x1160-1400` floor ending in a cliff
  edge at `x1400`, then open air down to `y=720` (a ~280px fall — over
  `MAX_FALL = 250`, so this drop is genuinely fatal without FLOATER,
  making the lesson meaningful, not just illustrative).
- Floor at `y=720`: `x1400-1650` floor, then a fork at `x1650` — the
  straight-ahead branch leads to a dead-end pit, the correct branch
  (reached by a BLOCKER redirecting the group) climbs slightly to
  continue toward the final obstacle.
- `x1750-1840`: a final solid wall (BOMBER blasts through it) leading to
  `EXIT` at `x1950, y700`.

This is a starting layout — Task 1's `gateX` values assume it, but the
mandatory screenshot verification below is expected to surface spots that
need coordinate tweaks (e.g. a slope that doesn't connect cleanly, a wall
that's climbable from the wrong side). Adjust coordinates in both this
task and, if an obstacle's actual cleared x-position shifts, Task 1's
`TUTORIAL_SEQUENCE[i].gateX` to match.

- [ ] **Step 1: Add the paintTutorialTerrain() method skeleton and obstacle 1 (BUILDER gap)**

Find this exact literal text (right after `paintTerrain()`'s closing
brace, before the next method — use the exact end of `paintTerrain()`
shown in this plan's Global Constraints research as the anchor; the
closing sequence is):

```
c.lineTo(W, 490); c.lineTo(1032, 492); c.closePath(); c.fill(); c.stroke();\n  }\n
```

Replace it with:

```
c.lineTo(W, 490); c.lineTo(1032, 492); c.closePath(); c.fill(); c.stroke();\n  }\n  paintTutorialTerrain() {\n    const c = this.tc, W = this.W, H = this.H;\n    c.clearRect(0, 0, W, H);\n    const seg1 = (cc) => {\n      cc.moveTo(0, 300); cc.lineTo(200, 300); cc.lineTo(200, H); cc.lineTo(0, H); cc.closePath();\n    };\n    this.terraShape(c, seg1, \"#C89D63\");\n  }\n
```

This paints only the very first floor segment (up to the BUILDER gap at
`x200`) so the very first render already shows something real to verify.

- [ ] **Step 2: Verify obstacle 1 renders**

With the server from Task 1 running, reload and click TUTORIAL. Confirm
a short floor segment appears from the left edge to about `x200`, then
open space (the intentional gap) — no JS console errors. Look at the
rendered canvas (via your environment's browser/screenshot tooling) to
confirm the shape looks like a sensible starting floor, not a stray
shape or an unfilled/misaligned region.

- [ ] **Step 3: Add obstacle 1's far side and obstacle 2 (CLIMBER wall)**

Find this exact literal text (the `seg1` block just added):

```
const seg1 = (cc) => {\n      cc.moveTo(0, 300); cc.lineTo(200, 300); cc.lineTo(200, H); cc.lineTo(0, H); cc.closePath();\n    };\n    this.terraShape(c, seg1, \"#C89D63\");\n
```

Replace it with:

```
const seg1 = (cc) => {\n      cc.moveTo(0, 300); cc.lineTo(200, 300); cc.lineTo(200, H); cc.lineTo(0, H); cc.closePath();\n    };\n    this.terraShape(c, seg1, \"#C89D63\");\n    const seg2 = (cc) => {\n      cc.moveTo(280, 300); cc.lineTo(420, 300); cc.lineTo(420, 140); cc.lineTo(460, 140); cc.lineTo(460, H); cc.lineTo(280, H); cc.closePath();\n    };\n    this.terraShape(c, seg2, \"#C89D63\");\n
```

`seg2` covers the floor from `x280` (far side of the BUILDER gap) to
`x420`, then the CLIMBER wall rising to `y140` at `x420-460`, filled all
the way down to `H` (solid ground beneath, same as Moss Hollow's floor).

- [ ] **Step 4: Verify obstacle 2 renders**

Reload the tutorial. Confirm the gap from `x200-280` is still open air,
floor resumes at `x280`, and a tall wall face is visible at `x420-460`
rising to about a third of the canvas height. Visually confirm the wall
looks climbable (a clean vertical face, not a jagged/broken edge).

- [ ] **Step 5: Add obstacle 3 (BASHER pillar) and connecting floor**

Find this exact literal text (the `seg2` block just added):

```
const seg2 = (cc) => {\n      cc.moveTo(280, 300); cc.lineTo(420, 300); cc.lineTo(420, 140); cc.lineTo(460, 140); cc.lineTo(460, H); cc.lineTo(280, H); cc.closePath();\n    };\n    this.terraShape(c, seg2, \"#C89D63\");\n
```

Replace it with:

```
const seg2 = (cc) => {\n      cc.moveTo(280, 300); cc.lineTo(420, 300); cc.lineTo(420, 140); cc.lineTo(460, 140); cc.lineTo(460, H); cc.lineTo(280, H); cc.closePath();\n    };\n    this.terraShape(c, seg2, \"#C89D63\");\n    const seg3 = (cc) => {\n      cc.moveTo(460, 140); cc.lineTo(600, 140); cc.lineTo(600, 100); cc.lineTo(640, 100); cc.lineTo(640, H); cc.lineTo(460, H); cc.closePath();\n    };\n    this.terraShape(c, seg3, \"#C89D63\");\n
```

`seg3` continues the `y140` floor from `x460-600`, then the BASHER
pillar rises further to `y100` at `x600-640`.

- [ ] **Step 6: Verify obstacle 3 renders**

Reload. Confirm floor continues at the elevated `y140` level after the
climber wall, and a narrower pillar rises above that floor at
`x600-640`. Confirm the pillar reads as a distinct obstacle to tunnel
through (roughly waist-to-head height above the `y140` floor line), not
so tall it looks unreachable.

- [ ] **Step 7: Add obstacle 4 (MINER slope) and connecting floor**

Find this exact literal text (the `seg3` block just added):

```
const seg3 = (cc) => {\n      cc.moveTo(460, 140); cc.lineTo(600, 140); cc.lineTo(600, 100); cc.lineTo(640, 100); cc.lineTo(640, H); cc.lineTo(460, H); cc.closePath();\n    };\n    this.terraShape(c, seg3, \"#C89D63\");\n
```

Replace it with:

```
const seg3 = (cc) => {\n      cc.moveTo(460, 140); cc.lineTo(600, 140); cc.lineTo(600, 100); cc.lineTo(640, 100); cc.lineTo(640, H); cc.lineTo(460, H); cc.closePath();\n    };\n    this.terraShape(c, seg3, \"#C89D63\");\n    const seg4 = (cc) => {\n      cc.moveTo(640, 140); cc.lineTo(780, 140); cc.lineTo(950, 300); cc.lineTo(1100, 300); cc.lineTo(1100, H); cc.lineTo(640, H); cc.closePath();\n    };\n    this.terraShape(c, seg4, \"#C89D63\");\n
```

`seg4` continues the `y140` floor from `x640-780`, then a diagonal slope
down to `y300` between `x780-950` (MINER digs through this mass), then
floor at `y300` from `x950-1100`.

- [ ] **Step 8: Verify obstacle 4 renders**

Reload. Confirm the floor after the basher pillar continues at `y140`,
then a diagonal solid slope descends to the lower `y300` level, and
floor continues there. Confirm the slope's angle looks traversable (not
a vertical cliff, not so shallow it's indistinguishable from flat
ground).

- [ ] **Step 9: Add obstacle 5 (DIGGER floating slab + lower corridor start)**

Find this exact literal text (the `seg4` block just added):

```
const seg4 = (cc) => {\n      cc.moveTo(640, 140); cc.lineTo(780, 140); cc.lineTo(950, 300); cc.lineTo(1100, 300); cc.lineTo(1100, H); cc.lineTo(640, H); cc.closePath();\n    };\n    this.terraShape(c, seg4, \"#C89D63\");\n
```

Replace it with:

```
const seg4 = (cc) => {\n      cc.moveTo(640, 140); cc.lineTo(780, 140); cc.lineTo(950, 300); cc.lineTo(1100, 300); cc.lineTo(1100, H); cc.lineTo(640, H); cc.closePath();\n    };\n    this.terraShape(c, seg4, \"#C89D63\");\n    const slab = (cc) => {\n      cc.moveTo(1100, 300); cc.lineTo(1160, 300); cc.lineTo(1160, 340); cc.lineTo(1100, 340); cc.closePath();\n    };\n    this.terraShape(c, slab, \"#C89D63\");\n    const seg5 = (cc) => {\n      cc.moveTo(1160, 440); cc.lineTo(1400, 440); cc.lineTo(1400, H); cc.lineTo(1160, H); cc.closePath();\n    };\n    this.terraShape(c, seg5, \"#C89D63\");\n
```

`slab` is the thin floating floor (`y300-340` only, `x1100-1160`) DIGGER
must dig straight down through. `seg5` is the lower corridor floor at
`y440`, from `x1160` (directly below the slab) to `x1400` where it ends
in a cliff edge — deliberately NOT connected to `seg4`/`slab` by any
solid fill above `y340` between them, so the only way down is digging
through the slab.

- [ ] **Step 10: Verify obstacle 5 renders**

Reload. Confirm a thin floating floor segment appears at `x1100-1160`
with visible open space both above (already-open walking area) and below
it. Confirm a separate lower floor appears starting around `x1160` at
roughly `y440`, disconnected from anything above it — this is the
digger's target: dig down from the upper level, land on this lower
floor.

- [ ] **Step 11: Add obstacle 6 (FLOATER cliff + fall) and lower corridor continuation**

Find this exact literal text (the `seg5` block just added):

```
const seg5 = (cc) => {\n      cc.moveTo(1160, 440); cc.lineTo(1400, 440); cc.lineTo(1400, H); cc.lineTo(1160, H); cc.closePath();\n    };\n    this.terraShape(c, seg5, \"#C89D63\");\n
```

Replace it with:

```
const seg5 = (cc) => {\n      cc.moveTo(1160, 440); cc.lineTo(1400, 440); cc.lineTo(1400, H); cc.lineTo(1160, H); cc.closePath();\n    };\n    this.terraShape(c, seg5, \"#C89D63\");\n    const seg6 = (cc) => {\n      cc.moveTo(1500, 720); cc.lineTo(1650, 720); cc.lineTo(1650, H); cc.lineTo(1500, H); cc.closePath();\n    };\n    this.terraShape(c, seg6, \"#C89D63\");\n
```

`seg6` is the floor at the bottom of the fall (`y720`), starting at
`x1500` — deliberately not connected to `seg5`'s `x1400` cliff edge by
any solid fill in between (a ~280px vertical gap, over `MAX_FALL=250`,
matching the terrain-layout note above), so the drop between them is a
genuine fall the plod must survive via FLOATER.

- [ ] **Step 12: Verify obstacle 6 renders**

Reload. Confirm `seg5`'s floor ends in a clean cliff edge around `x1400`,
open air continues well below it, and a separate floor segment starts
around `x1500` much lower on the canvas (`y720`). Confirm the vertical
gap between the cliff edge and the lower floor looks like a serious fall,
not a minor step.

- [ ] **Step 13: Add obstacle 7 (BLOCKER fork) and obstacle 8 (BOMBER wall) plus EXIT area**

Find this exact literal text (the `seg6` block just added):

```
const seg6 = (cc) => {\n      cc.moveTo(1500, 720); cc.lineTo(1650, 720); cc.lineTo(1650, H); cc.lineTo(1500, H); cc.closePath();\n    };\n    this.terraShape(c, seg6, \"#C89D63\");\n
```

Replace it with:

```
const seg6 = (cc) => {\n      cc.moveTo(1500, 720); cc.lineTo(1650, 720); cc.lineTo(1650, H); cc.lineTo(1500, H); cc.closePath();\n    };\n    this.terraShape(c, seg6, \"#C89D63\");\n    const deadEnd = (cc) => {\n      cc.moveTo(1650, 720); cc.lineTo(1750, 720); cc.lineTo(1750, 760); cc.lineTo(1650, 760); cc.closePath();\n    };\n    this.terraShape(c, deadEnd, \"#C89D63\");\n    const safePath = (cc) => {\n      cc.moveTo(1650, 700); cc.lineTo(1750, 660); cc.lineTo(1750, H); cc.lineTo(1650, H); cc.closePath();\n    };\n    this.terraShape(c, safePath, \"#C89D63\");\n    const wall = (cc) => {\n      cc.moveTo(1750, 660); cc.lineTo(1840, 660); cc.lineTo(1840, 500); cc.lineTo(1900, 500); cc.lineTo(1900, H); cc.lineTo(1750, H); cc.closePath();\n    };\n    this.terraShape(c, wall, \"#C89D63\");\n    const exitPad = (cc) => {\n      cc.moveTo(1900, 700); cc.lineTo(W, 700); cc.lineTo(W, H); cc.lineTo(1900, H); cc.closePath();\n    };\n    this.terraShape(c, exitPad, \"#C89D63\");\n  }\n
```

`deadEnd` is a short low ledge past the fork (`x1650-1750`, `y720-760`)
that dead-ends into a drop — the "wrong" branch a plod walking straight
ahead would take. `safePath` is a short rise (`x1650-1750`, up to
`y660`) — the branch a BLOCKER-redirected plod takes instead. `wall` is
the final BOMBER obstruction (`x1750-1840` rising to `y500`, plus a
short connecting lip to `x1900`). `exitPad` is flat ground under `EXIT`
(`x1900` to `W`, at `y700`, matching `TUTORIAL_LEVEL.EXIT = {x:1950,
y:700}` from Task 1).

- [ ] **Step 14: Verify obstacle 7 and 8 render, and the full level is complete**

Reload. Confirm: a fork exists around `x1650` with a low dead-end ledge
and a slightly higher continuing path; a solid wall blocks the path
around `x1750-1840`; flat ground exists under where `EXIT` is
positioned. Walk the whole rendered shape visually end-to-end (left to
right) and confirm there are no unreachable/floating disconnected
shapes other than the two intentional gaps (`x200-280` BUILDER gap,
`x1400-1500` FLOATER fall) and the intentional under-slab hollow
(DIGGER).

- [ ] **Step 15: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add index.html
git commit -m "feat(tutorial): add paintTutorialTerrain() with 8 obstacles (#4)"
```

---

### Task 3: Skill-locking — gate assign() and dim non-current-step tiles

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `state.levelId`, `this.tutorialStep`, `TUTORIAL_SEQUENCE` from
  Tasks 1-2.
- Produces: modifies `assign()` and `renderVals()`'s tile-dimming logic in
  place — no new methods.

- [ ] **Step 1: Gate assign() to the current tutorial step's skill**

Find this exact literal text (the start of `assign()`):

```
assign(p, key) {\n    if (this.counts[key] <= 0) return;\n    let ok = false;\n
```

Replace it with:

```
assign(p, key) {\n    if (this.counts[key] <= 0) return;\n    if (this.state.levelId === \"tutorial\" && this.tutorialStep != null && this.tutorialStep < this.TUTORIAL_SEQUENCE.length && key !== this.TUTORIAL_SEQUENCE[this.tutorialStep].skill) return;\n    let ok = false;\n
```

This rejects any assignment attempt for a skill other than the current
step's — the same early-return pattern already used for the zero-count
check right above it.

- [ ] **Step 2: Extend the tile-dimming condition**

Find this exact literal text (inside `renderVals()`'s `tiles` mapping):

```
const sel = s.selected === sk.key, n = counts[sk.key] ?? sk.n, dim = n <= 0;\n
```

Replace it with:

```
const tutLocked = this.state.levelId === \"tutorial\" && this.tutorialStep != null && this.tutorialStep < this.TUTORIAL_SEQUENCE.length && sk.key !== this.TUTORIAL_SEQUENCE[this.tutorialStep].skill;\n      const sel = s.selected === sk.key, n = counts[sk.key] ?? sk.n, dim = n <= 0 || tutLocked;\n
```

Reuses the exact same `opacity: dim ? 0.42 : 1` / `boxShadow: dim ?
"none" : ...` visual treatment already wired to `dim` — no new CSS
needed.

- [ ] **Step 3: Manual verification**

With the tutorial's terrain now in place from Task 2, reload and click
TUTORIAL. Confirm only the BUILDER tile is at full opacity/clickable;
all 7 others are dimmed. Click a dimmed tile (e.g. DIGGER) and confirm
nothing happens (no skill gets selected/assigned). This step doesn't yet
have a way to advance past step 0 (that's Task 4) — confirming the lock
itself is sufficient here.

- [ ] **Step 4: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add index.html
git commit -m "feat(tutorial): lock skill tiles to the current tutorial step (#4)"
```

---

### Task 4: Prompt banner and step-advance logic

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `state.levelId`, `this.tutorialStep`, `TUTORIAL_SEQUENCE` from
  Tasks 1-3.
- Produces: a `tutorialPrompt` getter (rendered via a new `{{ }}` binding),
  a `showTutorialBanner` getter, step-advance logic inside `update(dt)`.

- [ ] **Step 1: Add step-advance logic to update()**

Find this exact literal text (the end of `update(dt)`'s plod loop, right
before the particle-update code):

```
this.plods = this.plods.filter((p) => !dead.includes(p));\n    // particles\n
```

Replace it with:

```
this.plods = this.plods.filter((p) => !dead.includes(p));\n    if (this.state.levelId === \"tutorial\" && this.tutorialStep != null && this.tutorialStep < this.TUTORIAL_SEQUENCE.length) {\n      const gateX = this.TUTORIAL_SEQUENCE[this.tutorialStep].gateX;\n      if (this.plods.some((p) => p.x > gateX)) this.tutorialStep++;\n    }\n    // particles\n
```

This runs once per simulated tick (only while `phase === "play"`, after
all plods have updated for that tick) — checked here specifically
because gate-passing depends on post-update plod positions, matching the
Global Constraints/Architecture note on why this is the right hook point
rather than the top of `tick(dt)`.

- [ ] **Step 2: Add the prompt banner markup**

Find this exact literal text (the HUD strip's closing area — reuse the
FLOW tile's container as an anchor, right after it, so the banner sits
below the whole HUD row without disturbing existing HUD markup; anchor on
the exact FLOW closing sequence):

```
color:#8A8072\">FLOW<\u002Fspan><span style=\"font:700 20px 'Space Mono',monospace;color:#2B2320\">{{ flow }}<\u002Fspan><\u002Fdiv>\n
```

Replace it with:

```
color:#8A8072\">FLOW<\u002Fspan><span style=\"font:700 20px 'Space Mono',monospace;color:#2B2320\">{{ flow }}<\u002Fspan><\u002Fdiv>\n  <sc-if value=\"{{ showTutorialBanner }}\" hint-placeholder-val=\"{{ false }}\">\n    <div style=\"position:absolute;top:78px;left:50%;transform:translateX(-50%);background:#FFFBF2;border:2.5px solid #2E7CD6;border-radius:12px;box-shadow:0 4px 0 rgba(43,35,32,.22);padding:10px 20px;font:500 14px Fredoka,sans-serif;color:#2B2320;z-index:6;white-space:nowrap\">{{ tutorialPrompt }}<\u002Fdiv>\n  <\u002Fsc-if>\n
```

`top:78px` sits just below the HUD strip (which sits at the very top of
the play area) without overlapping it, per the spec's "not blocking the
play area" requirement — verify this visually in Step 4 and adjust if it
overlaps.

- [ ] **Step 3: Add the tutorialPrompt/showTutorialBanner getters**

Find this exact literal text (the `flowLabel`-equivalent area doesn't
exist pre-i18n — anchor instead on the `nukeLabel` ternary, which is a
stable pre-i18n anchor already used successfully in issue #5's plan):

```
nukeLabel: s.nukeArm ? \"SURE?\" : \"NUKE\",\n
```

Replace it with:

```
nukeLabel: s.nukeArm ? \"SURE?\" : \"NUKE\",\n      showTutorialBanner: this.state.levelId === \"tutorial\" && this.tutorialStep != null && this.tutorialStep < this.TUTORIAL_SEQUENCE.length,\n      tutorialPrompt: this.state.levelId === \"tutorial\" && this.tutorialStep != null && this.tutorialStep < this.TUTORIAL_SEQUENCE.length ? this.TUTORIAL_SEQUENCE[this.tutorialStep].prompt : \"\",\n
```

- [ ] **Step 4: Manual verification**

With the server running, click TUTORIAL. Confirm a banner appears below
the HUD showing step 1's prompt text ("Click BUILDER, then click a plod
— bridge the gap."), doesn't overlap the HUD strip or skill tiles, and
disappears/updates to the next prompt once a plod's x passes the
BUILDER gate (~x300) — verify by actually assigning BUILDER to a plod
and guiding it across the gap. Confirm CLIMBER unlocks (full opacity)
the moment the banner updates to step 2's text, matching Task 3's
locking logic.

- [ ] **Step 5: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add index.html
git commit -m "feat(tutorial): add prompt banner and gate-based step advancement (#4)"
```

---

### Task 5: Tutorial completion overlay

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `this.tutorialStep`, `TUTORIAL_SEQUENCE.length`, `this.saved`
  from Tasks 1-4.
- Produces: a `"tutorialDone"` phase value, a `showTutorialDone` getter, a
  matching `<sc-if>` overlay block, and a guard preventing the tutorial
  from ever reaching the normal `won`/`lost` phases.

- [ ] **Step 1: Guard the normal finish() auto-triggers for the tutorial**

Find this exact literal text (inside `tick(dt)`):

```
if (this.timeLeft <= 0) this.finish();\n      if (this.spawned >= this.total() && this.plods.length === 0) this.finish();\n
```

Replace it with:

```
if (this.state.levelId !== \"tutorial\") {\n        if (this.timeLeft <= 0) this.finish();\n        if (this.spawned >= this.total() && this.plods.length === 0) this.finish();\n      } else if (this.tutorialStep != null && this.tutorialStep >= this.TUTORIAL_SEQUENCE.length && this.saved > 0 && this.state.phase !== \"tutorialDone\") {\n        this.setState({ phase: \"tutorialDone\" });\n      }\n
```

The tutorial never calls the normal `finish()` (so it can never resolve
to `won`/`lost`); instead it transitions straight to a new `tutorialDone`
phase once all 8 steps are complete (`tutorialStep` has incremented past
the last index) and at least one plod has reached home.

- [ ] **Step 2: Add the tutorial-complete overlay markup**

Find this exact literal text (the TRY AGAIN button through the end of
the lost screen's `<sc-if>` and the overlay's closing wrapper divs):

```
sc-camel-on-click=\"{{ restart }}\">TRY AGAIN<\u002Fdiv>\n  <\u002Fsc-if>\n<\u002Fdiv>\n<\u002Fdiv>\n<\u002Fsc-if>\n<\u002Fdiv>\n<\u002Fdiv>\n<\u002Fdiv>\n
```

Replace it with:

```
sc-camel-on-click=\"{{ restart }}\">TRY AGAIN<\u002Fdiv>\n  <\u002Fsc-if>\n  <sc-if value=\"{{ showTutorialDone }}\" hint-placeholder-val=\"{{ false }}\">\n    <div style=\"font:600 26px Fredoka,sans-serif;color:#2B2320\">YOU'RE READY!<\u002Fdiv>\n    <div style=\"font:400 12px 'Space Mono',monospace;color:#6E6255\">You've learned all 8 skills. Time for the real thing.<\u002Fdiv>\n    <div style=\"width:220px;height:60px;background:#FFFBF2;border:3px solid #2E7CD6;border-radius:13px;box-shadow:0 5px 0 rgba(43,35,32,.24);display:grid;place-items:center;font:600 20px Fredoka,sans-serif;color:#2B2320;cursor:pointer\" sc-camel-on-click=\"{{ backToMenu }}\">BACK TO MENU<\u002Fdiv>\n  <\u002Fsc-if>\n<\u002Fdiv>\n<\u002Fdiv>\n<\u002Fsc-if>\n<\u002Fdiv>\n<\u002Fdiv>\n<\u002Fdiv>\n
```

The new tutorial-done `<sc-if>` block is inserted as a sibling right
after the lost screen's closing `<\u002Fsc-if>`, before the overlay's own
wrapper divs close — so it's another peer screen inside the same overlay
container, exactly like `showWon`/`showLost` are peers of each other.

- [ ] **Step 3: Add showTutorialDone and backToMenu getters, and phase→overlay wiring**

Find this exact literal text (the `showOverlay`/`showReady`/etc. getters):

```
showOverlay: s.phase !== \"play\", showReady: s.phase === \"ready\", showPaused: s.phase === \"paused\", showWon: s.phase === \"won\", showLost: s.phase === \"lost\",\n
```

Replace it with:

```
showOverlay: s.phase !== \"play\", showReady: s.phase === \"ready\", showPaused: s.phase === \"paused\", showWon: s.phase === \"won\", showLost: s.phase === \"lost\", showTutorialDone: s.phase === \"tutorialDone\",\n      backToMenu: () => this.setState({ phase: \"ready\", levelId: \"moss\" }, () => this.initLevel()),\n
```

`showOverlay` already covers `tutorialDone` for free (`s.phase !== "play"`
is true for it), so the overlay container itself needs no change —
only the new inner `<sc-if>` block from Step 2. `backToMenu` resets back
to Moss Hollow's ready screen (matching Task 1's `startGame` pattern of
setting `levelId` then re-running `initLevel()`), so PLAY/TUTORIAL are
both available again afterward.

- [ ] **Step 4: Manual verification**

With the server running, play through the full tutorial from TUTORIAL to
completion (using the prompts/gates from Tasks 2-4). Confirm: no timer
ever appears/counts down, no lose condition triggers regardless of how
long it takes, and upon reaching the BOMBER gate with a plod having
reached EXIT/home, the new "YOU'RE READY!" screen appears (not the
normal win screen) with a working BACK TO MENU button that returns to
the ready screen with PLAY/TUTORIAL both available.

- [ ] **Step 5: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add index.html
git commit -m "feat(tutorial): add tutorial-complete overlay, guard normal win/lose (#4)"
```

---

### Task 6: Full regression pass, CHANGELOG, and push

**Files:**
- Modify: `CHANGELOG.md`

**Interfaces:**
- Consumes: nothing new.
- Produces: nothing new.

- [ ] **Step 1: Full manual regression pass**

With the server running:
1. Confirm Moss Hollow (`PLAY`) is completely unaffected — same terrain,
   timer, quota, skill counts, win/lose screens as before this feature.
2. Play the full tutorial start to finish. Confirm each of the 8 steps
   locks to its named skill only, the banner text matches
   `TUTORIAL_SEQUENCE`, and each gate advances the step only after the
   correct obstacle is actually cleared (not prematurely).
3. Confirm the FLOATER obstacle (step 6) is genuinely fatal to a plod
   without floater assigned before the drop (spot-check: assign floater
   to one plod, don't assign it to a second, confirm the second dies
   from the fall while the first survives) — this validates the
   MAX_FALL-exceeding drop distance from Task 2.
4. Confirm `R` (restart) while in the tutorial restarts the tutorial
   (not Moss Hollow), and restarting while in Moss Hollow restarts Moss
   Hollow.
5. Confirm BACK TO MENU from the tutorial-complete screen, and playing
   Moss Hollow immediately afterward without a page reload, both work
   correctly (no stale `W`/`H`/`HATCH`/`EXIT` leaking between levels).

- [ ] **Step 2: Add changelog entry**

Add under the `[Unreleased]` section's `Added` subsection (create the
subsection if it doesn't exist yet) in `CHANGELOG.md`:

```markdown
### Added

- Guided tutorial level teaching all 8 skills in sequence, reached via a
  new TUTORIAL button on the ready screen — untimed, no fail state (#4).
```

- [ ] **Step 3: Commit and push**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add CHANGELOG.md
git commit -m "docs(changelog): note guided tutorial level (#4)"
git push
```
