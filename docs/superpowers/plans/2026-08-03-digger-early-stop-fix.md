# Digger/Miner Early-Stop Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the digger (issue #5) and miner plodder states so they only stop
digging when they hit steel, reach the map bottom, or find a genuine open gap
below — not immediately after every carve.

**Architecture:** Single-file change to `index.html`'s embedded plodder
state-machine. Each fix replaces a same-y-level "is the ground beside my feet
empty" probe with a probe placed past the radius of the carve the plodder just
made, so the check tests for real open space below the freshly-dug cavity
instead of the (expected, harmless) hollow the dig itself created.

**Tech Stack:** Vanilla JS, single self-contained `index.html`, no build step,
no test framework.

## Global Constraints

- `index.html` is normally a generated dc-tool bundle (per this project's
  `game-*` conventions, bundled games should have source `.dc.html` files
  edited and re-bundled, never the generated output). No source file exists
  in this repo or its git history for game-plod — confirmed with the user,
  who approved hand-editing `index.html` directly for this fix. Do not go
  looking for a source file elsewhere; there isn't one to find.
- The plodder logic in `index.html` is stored as a JSON-escaped string
  inside an HTML attribute: newlines in the source are the literal two-byte
  sequence `\n` (backslash + n) and double quotes are `\"` (backslash +
  quote), not actual newline/quote characters. Every code snippet in this
  plan uses those literal escaped sequences — copy them exactly as shown,
  do not "unescape" them.
- No automated test suite exists in this repo. Every task's verification
  step is a manual playtest using a local static server.
- Surgical edits only — don't touch unrelated branches (`basher`, `builder`,
  etc.) or reformat surrounding code.

---

### Task 1: Fix digger early-stop bug

**Files:**
- Modify: `index.html` (single embedded script; locate via the exact string
  match below — there are no useful line numbers in this file)

**Interfaces:**
- Consumes: existing instance methods `this.carve(x, y, r)`, `this.solid(x, y)`,
  `this.gridAt(x, y)`, and `this.H` (map height in px) — all already used
  elsewhere in this file, unchanged by this task.
- Produces: n/a (terminal behavior fix, no new interface for other tasks to
  consume).

- [ ] **Step 1: Locate and replace the digger branch**

Find this exact literal text in `index.html` (it appears once, immediately
after the `climber` branch and before the `basher` branch):

```
} else if (S === \"digger\") {\n      p.t += dt;\n      if (p.t > 0.13) { p.t = 0; this.carve(p.x, p.y + 7, 15); p.y += 3; this.poof(p.x + (Math.random() * 30 - 15), p.y + 4, \"#EADFC9\", 2); if (!this.solid(p.x - 7, p.y + 2) && !this.solid(p.x + 7, p.y + 2)) { p.state = \"faller\"; p.vy = 30; p.fallY = p.y; } if (this.gridAt(p.x, p.y + 4) === 2) p.state = \"walker\"; }\n    }
```

Replace it with:

```
} else if (S === \"digger\") {\n      p.t += dt;\n      if (p.t > 0.13) { p.t = 0; const cx = p.x, cy = p.y + 7, cr = 15; this.carve(cx, cy, cr); p.y += 3; this.poof(p.x + (Math.random() * 30 - 15), p.y + 4, \"#EADFC9\", 2); if (this.gridAt(p.x, p.y + 4) === 2) { p.state = \"walker\"; } else if (p.y > this.H - 4) { p.state = \"walker\"; } else if (!this.solid(cx, cy + cr + 4)) { p.state = \"faller\"; p.vy = 30; p.fallY = p.y; } }\n    }
```

What changed:
- `cx`/`cy`/`cr` capture the exact center and radius passed to `carve(...)`,
  so the later check is provably tied to the cavity that was actually dug.
- The old sideways probe (`!solid(x-7,y+2) && !solid(x+7,y+2)`, at roughly
  the same y-level as the dig point) is replaced with a single downward
  probe at `cy + cr + 4` — 4px past the bottom edge of the carved circle.
  Right after a carve, that point is normally still solid ground; it only
  reads empty when there's a genuine gap below the newly-dug hole.
  - `4` is a small clearance margin past the exact carve radius, matching
    the margin used elsewhere in this file for similar probes (e.g. the
    `p.y + 4` gap in the steel check on the same line).
- A new `p.y > this.H - 4` bottom-of-map check is added, mirroring the
  miner's existing pattern, and is checked before the open-space probe so
  reaching the bottom always resolves to `walker`.
- Check order is steel → bottom → open-space, so a spot that satisfies more
  than one condition resolves the same way a human reading "hits something,
  reaches the bottom, or finds a gap" would expect.

- [ ] **Step 2: Manual verification — narrow shaft (regression case)**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
python3 -m http.server 8123
```

Open `http://localhost:8123/index.html` in a browser. Select a plodder,
assign the digger skill (hotkey `8`), and place it digging straight down
through a narrow column of terrain (pick a spot where the terrain to either
side of the dig path is thin). Expected: the plodder keeps digging through
the full column instead of switching to `faller` after one or two carves.

- [ ] **Step 3: Manual verification — steel, bottom, and real gap**

With the same server running:
- Dig a digger into a steel/indestructible strip (grid value 2 areas, if
  present on the level) — expect it becomes `walker` immediately, unchanged
  from before.
- Dig a digger all the way to the bottom of the map — expect it becomes
  `walker` at the bottom instead of endlessly digging or glitching past the
  map edge.
- Dig a digger so that it digs out over a real pre-existing open area/void
  (e.g. dig two shafts side by side first, then dig a third that breaks
  into one of them) — expect it becomes `faller` once it reaches the real
  gap, confirming the fix doesn't just always keep it digging.

- [ ] **Step 4: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add index.html
git commit -m "fix(digger): stop only on steel, map bottom, or a real gap below (#5)"
```

---

### Task 2: Fix miner's identical early-stop bug

**Files:**
- Modify: `index.html` (same file, `miner` branch — immediately after the
  `basher` branch, before the `builder` branch)

**Interfaces:**
- Consumes: same instance methods as Task 1 (`this.carve`, `this.solid`,
  `this.gridAt`, `this.H`), plus `p.dir` (already used by miner for its
  forward-digging direction).
- Produces: n/a.

- [ ] **Step 1: Locate and replace the miner branch**

Find this exact literal text in `index.html` (appears once, inside the
`miner` branch, after the steel check and before the `builder` branch):

```
this.carve(p.x + p.dir * 12, p.y - 5, 17); p.x += p.dir * 3.4; p.y += 2.8;\n        this.spark(p.x + p.dir * 14, p.y - 4);\n        if (!this.solid(p.x, p.y + 3) && !this.solid(p.x + p.dir * 8, p.y + 3)) { p.state = \"faller\"; p.vy = 40; p.fallY = p.y; }\n        if (p.y > this.H - 4) { p.state = \"walker\"; }
```

Replace it with:

```
const cx = p.x + p.dir * 12, cy = p.y - 5, cr = 17; this.carve(cx, cy, cr); p.x += p.dir * 3.4; p.y += 2.8;\n        this.spark(p.x + p.dir * 14, p.y - 4);\n        if (p.y > this.H - 4) { p.state = \"walker\"; } else if (!this.solid(cx, cy + cr + 4)) { p.state = \"faller\"; p.vy = 40; p.fallY = p.y; }
```

What changed: same shape of fix as Task 1, scaled to the miner's own carve
call (`carve(p.x + p.dir*12, p.y-5, 17)` instead of digger's
`carve(p.x, p.y+7, 15)`) — `cx`/`cy`/`cr` capture that call's actual
arguments, and the sideways probe is replaced with a downward probe at
`cy + cr + 4`. The miner's existing steel check
(`gridAt(p.x + p.dir*14, p.y) === 2`, earlier in the same branch, not shown
above) is unchanged. Bottom check now runs before the open-space check, same
ordering rationale as Task 1.

- [ ] **Step 2: Manual verification — narrow shaft / diagonal tunnel**

With the same local server from Task 1 running (`python3 -m http.server 8123`
from the repo root, if not already running), assign the miner skill (hotkey
`7`) to a plodder in a spot where it tunnels diagonally through a thin
strip of terrain. Expected: it keeps mining through the full strip instead
of switching to `faller` after one or two carves.

- [ ] **Step 3: Manual verification — steel, bottom, and real gap**

- Mine into a steel strip — expect immediate `walker`, unchanged from before.
- Mine to the map bottom — expect `walker` at the bottom.
- Mine so it breaks into a genuinely open pre-existing area — expect
  `faller` once it reaches the real gap.

- [ ] **Step 4: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add index.html
git commit -m "fix(miner): stop only on steel, map bottom, or a real gap below"
```

---

### Task 3: Update CHANGELOG and push

**Files:**
- Modify: `CHANGELOG.md`

**Interfaces:**
- Consumes: n/a
- Produces: n/a

- [ ] **Step 1: Add changelog entry**

Add under the `[Unreleased]` section's `Fixed` subsection (create the
subsection if it doesn't exist yet) in `CHANGELOG.md`:

```markdown
### Fixed

- Digger and miner no longer stop early right after carving — they now
  keep going until they hit steel, reach the map bottom, or find a real
  gap below (#5).
```

- [ ] **Step 2: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add CHANGELOG.md
git commit -m "docs(changelog): note digger/miner early-stop fix (#5)"
```

- [ ] **Step 3: Push**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git push
```
