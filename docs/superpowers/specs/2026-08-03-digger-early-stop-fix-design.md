# Digger (and miner) stops digging early — design

Issue: [freaxnx01/game-plod#5](https://github.com/freaxnx01/game-plod/issues/5)

## Problem

The digger plodder switches out of its `"digger"` state into `"faller"` far too
early — often after a single carve. The current stop-check is:

```js
if (!this.solid(p.x - 7, p.y + 2) && !this.solid(p.x + 7, p.y + 2)) {
  p.state = "faller"; p.vy = 30; p.fallY = p.y;
}
```

This probes ±7px to either side of the feet at roughly the same y-level the
plodder just carved. `carve(p.x, p.y + 7, 15)` hollows out a 15px-radius
circle around the dig point — a radius larger than the ±7px probe offset. So
immediately after carving, the probe reads "no solid ground" on both sides
almost every time, regardless of whether there's real ground further down.
The check is testing the wrong thing: "is there dirt beside my feet"
(usually false right after carving) instead of "is there solid ground below
the cavity I just dug" (the actual stop condition that matters).

The miner has the same shaped bug in its own side-probe:

```js
!this.solid(p.x, p.y + 3) && !this.solid(p.x + dir * 8, p.y + 3)
```

## Scope

- `index.html` only — the digger and miner branches of the plodder state
  machine (single embedded script, no build step, no separate modules).
- Both digger and miner get the corrected stop-check, since they share the
  same root-cause pattern.
- Player-initiated cancel is explicitly **out of scope**. No skill in the
  game currently supports cancelling mid-action; adding that is a
  cross-cutting feature, not part of this bug fix.

## Design

### Digger

Replace the sideways probe with a downward probe placed past the carve
radius, so it only fires when there is genuine open space below the
freshly-carved cavity:

```js
} else if (S === "digger") {
  p.t += dt;
  if (p.t > 0.13) {
    p.t = 0; this.carve(p.x, p.y + 7, 15); p.y += 3;
    this.poof(p.x + (Math.random()*30-15), p.y + 4, "#EADFC9", 2);
    if (this.gridAt(p.x, p.y + 4) === 2) {
      p.state = "walker";
    } else if (p.y > this.H - 4) {
      p.state = "walker"; // reached map bottom
    } else if (!this.solid(p.x, p.y + 15 + 4)) {
      p.state = "faller"; p.vy = 30; p.fallY = p.y; // genuine open space below the dug cavity
    }
  }
}
```

- `15` matches the carve radius passed to `carve(...)`; `+4` is a small
  margin so the probe lands just past the hollowed-out circle rather than
  exactly on its edge.
- Steel check (`gridAt(...)===2`) is unchanged — it already works correctly.
- Bottom-of-map check (`p.y > this.H - 4`) is new, mirroring the miner's
  existing pattern, and takes priority over the faller transition so
  reaching the bottom always resolves to `walker`, not `faller`.
- Order matters: steel and bottom checks run before the open-space probe,
  so a spot that's both "at the bottom" and "open below" (edge of map)
  resolves to `walker`, matching "reaches the bottom" from the issue.

### Miner

Same shaped fix, scaled to the miner's own carve/dig geometry — replace the
`!solid(p.x,p.y+3) && !solid(p.x+dir*8,p.y+3)` side-probe with a downward
probe placed past whatever radius the miner's own carve call uses, keeping
the miner's existing steel check (`gridAt(p.x+dir*14,p.y)===2`) and
bottom-of-map check (`p.y > this.H - 4`) as-is.

## Testing

No automated test suite exists in this repo (single self-contained
`index.html`, no test infra). Verification is manual playtest, covering for
both digger and miner:

- Digging through a narrow vertical shaft (regression case for the original
  bug — should now dig all the way through instead of stopping after one
  carve).
- Digging immediately after a wide carve, to confirm the widened probe
  doesn't itself cause the same problem.
- Digging until the map bottom is reached (with map bottom check).
- Digging into steel/indestructible terrain — confirms unchanged.
- Digging over an actual real gap/void below (e.g. deliberately carved-out
  space, or a natural bottom pit) — confirms `faller` still triggers when it
  should.

## Out of scope

- Player-cancel mechanism for any skill (codebase-wide gap, not specific to
  this bug).
- Any change to the basher's stop logic (already scans ahead correctly with
  `!ahead`, unaffected by this bug pattern).
