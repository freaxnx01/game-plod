# Document what each skill/action does — design

Issue: [freaxnx01/game-plod#3](https://github.com/freaxnx01/game-plod/issues/3)

## Problem

`README.md`'s "Notes" section lists the hotkeys (`Controls: hotkeys 1–8
select skills, SPACE pauses, N (twice) nukes, R restarts`) but never
explains what each of the 8 skills actually does. Players (and the repo's
own `todo.md`) have had to guess, and specifically flagged confusion over
what BASHER is for.

## Scope

- `README.md` only. No changes to `index.html`, `identity.html`, or any
  in-game UI/tooltip.
- Deliberately kept separate from issue #4 ("Add hint system and tutorial
  for younger players") — #4 may build an interactive in-game explanation
  later, and can reference or adapt this README text rather than this issue
  duplicating that work now.
- Purpose-focused, one line per skill (not full mechanical detail) plus a
  short cross-cutting-rules note — matches the README's existing terse,
  bullet-style tone rather than introducing a heavier reference table.

## Design

A new `## Skills` section is added to `README.md`, placed after `## Contents`
and before `## Publish to GitHub Pages` — early enough to read as core
gameplay reference alongside the file's other "what is this" content,
rather than buried near the bottom.

Content (drafted from ground-truth mechanics read directly out of
`index.html`'s plodder state machine):

```markdown
## Skills

- **1 · CLIMBER** — scales walls too tall to just walk up.
- **2 · FLOATER** — survives any fall, no matter how far.
- **3 · BOMBER** — detonates after 3s, blasting a wide crater (fatal to the plod).
- **4 · BLOCKER** — plants itself and turns walking plods around — a wall of one.
- **5 · BUILDER** — lays a plank staircase to bridge gaps or climb up.
- **6 · BASHER** — tunnels straight through a horizontal obstruction at floor height, so plods can walk through.
- **7 · MINER** — cuts a diagonal tunnel downward through sloped terrain.
- **8 · DIGGER** — drops straight down through a floor via a vertical shaft.

Steel terrain can't be dug through by any skill. BLOCKER/BUILDER/BASHER/MINER/DIGGER can only be assigned to a plod that's currently walking. BASHER, MINER, and DIGGER stop on their own once they break through — no need to cancel them.
```

This directly answers the issue's folded-in question ("What is BASHER good
for?") via the BASHER line, distinguishing it from MINER (diagonal) and
DIGGER (vertical) by direction and use case — the actual distinguishing
factor found in the code (basher = horizontal tunnel at floor height for
walking through an obstruction; miner = diagonal descent through a slope;
digger = straight drop through a floor).

## Testing / verification

No automated test suite exists in this repo (single self-contained
`index.html`, no test framework — same situation confirmed for issues #5
and #2). This is a documentation-only change to `README.md`; verification
is a read-through: confirm the new section renders correctly as Markdown
(headings, bold, bullets) and that all 8 skills plus the cross-cutting note
are present and accurate against the mechanics documented above.

## Out of scope

- In-game tooltips, hint system, or tutorial (issue #4).
- `identity.html`.
- Any change to actual game mechanics/balance.
