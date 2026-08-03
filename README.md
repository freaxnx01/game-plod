# PLOD — Publish Package

A playable Lemmings-style prototype ("PLOD") plus its visual identity board, bundled as self-contained static HTML. **No build step, no dependencies — ready to serve as-is.**

## Contents

- `index.html` — the playable game (Moss Hollow level: pixel-destructible terrain, all 8 skills, full win/lose loop). Fully self-contained (~320 KB).
- `identity.html` — the art-direction / visual identity board. Links back to the game.
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

Ask Claude Code something like:

> Publish this folder to GitHub Pages. Create a public repo called `plod`, push index.html and identity.html, and enable Pages from the main branch root.

Or run the steps yourself:

```bash
cd publish_package_plod
git init && git add index.html identity.html README.md
git commit -m "PLOD playable"
gh repo create plod --public --source=. --push
gh api repos/{owner}/plod/pages -X POST -f "source[branch]=main" -f "source[path]=/"
```

The game will be live at `https://<user>.github.io/plod/`.

## Notes

- Both HTML files work offline and from `file://` — you can double-click to test locally.
- Fonts (Fredoka, Space Mono) are embedded in the bundles.
- Controls: hotkeys 1–8 select skills, SPACE pauses, N (twice) nukes, R restarts.
- The two pages cross-link (`index.html` ⇄ `identity.html`); keep both at the same path level.
