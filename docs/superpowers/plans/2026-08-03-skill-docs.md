# Skill Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `## Skills` section to `README.md` explaining what each of
the 8 plodder skills does, including a direct answer to "what is BASHER
good for."

**Architecture:** Single addition to `README.md` — no code changes.

**Tech Stack:** Markdown.

## Global Constraints

- `README.md` only. Do not touch `index.html`, `identity.html`, or any
  in-game UI.
- Purpose-focused, one line per skill, matching the README's existing
  terse bullet style — not a full mechanics writeup.
- Explicitly out of scope: in-game tooltips/hint system/tutorial (that's
  issue #4, a separate plan).

---

### Task 1: Add the Skills section to README.md

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing (terminal documentation change).

- [ ] **Step 1: Insert the new section**

Find this exact literal text in `README.md`:

```
- `src/` — original editable sources (Design Component HTML + `support.js` runtime). Only needed if you want to modify the design; **do not deploy `src/`** unless you want the editable versions online too.

## Publish to GitHub Pages (via Claude Code)
```

Replace it with:

```
- `src/` — original editable sources (Design Component HTML + `support.js` runtime). Only needed if you want to modify the design; **do not deploy `src/`** unless you want the editable versions online too.

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

## Publish to GitHub Pages (via Claude Code)
```

- [ ] **Step 2: Verify the Markdown renders correctly**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
grep -c "^## Skills$" README.md
```

Expected: `1` (the section header exists exactly once). Then read the file
and confirm by eye that all 8 bullets, the bold hotkey+name lead-in, and
the closing cross-cutting-rules paragraph are present and unbroken (no
stray Markdown syntax, no merged lines).

- [ ] **Step 3: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git add README.md
git commit -m "docs: explain what each skill does in README (#3)"
```

- [ ] **Step 4: Push**

```bash
cd /home/freax/repos/github/freaxnx01/public/game-plod
git push
```
