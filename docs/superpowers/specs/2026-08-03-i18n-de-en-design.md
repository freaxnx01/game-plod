# i18n: DE/EN support (default EN) — design

Issue: [freaxnx01/game-plod#2](https://github.com/freaxnx01/game-plod/issues/2)

## Problem

`index.html` has no i18n structure at all. All ~35-45 user-facing strings are
inline literals scattered across the custom `{{ }}` template system and plain
JS, with no language switcher, no persisted preference, and several HUD
buttons at fixed pixel widths sized only for the English text.

## Scope

- `index.html` only. `identity.html` (the visual-identity board, a separate
  page not part of gameplay) is explicitly **out of scope**.
- Covers every player-facing string enumerated below. Level name
  ("1·1 MOSS HOLLOW") is treated as a proper noun and is **not** translated,
  same in both languages — consistent with how level/place names are
  typically handled in localization.
- Draft German translations below aim for tone parity with the terse,
  playful English original, not certified professional copy. The
  implementer may refine exact wording during implementation as long as the
  key structure and button-width decisions below are preserved.

## Design

### Mechanism

A flat `STRINGS` dictionary plus a `t(key, vars?)` lookup helper, added near
the top of the embedded script:

```js
const STRINGS = {
  en: { play: "PLAY", resume: "RESUME", /* ...full table below... */ },
  de: { play: "SPIELEN", resume: "WEITER", /* ...full table below... */ },
};
function t(key, vars) {
  const s = (STRINGS[this.state.lang] && STRINGS[this.state.lang][key]) ?? STRINGS.en[key];
  return vars ? s.replace(/\{(\w+)\}/g, (_, k) => vars[k]) : s;
}
```

- `t()` falls back to `STRINGS.en[key]` if the current language is missing a
  key, so a partial translation never renders blank.
- `vars` supports simple `{token}` interpolation for strings that embed
  dynamic values (e.g. the ready screen's plod counts), matching how the
  template already interpolates `{{ quota }}`/`{{ total }}` elsewhere.

### Language state and persistence

- New state field `state.lang`, initialized once at startup:
  `localStorage.getItem('plod-lang') ?? (navigator.language.startsWith('de') ? 'de' : 'en')`.
- Any change to `state.lang` (via the manual toggle) writes back to
  `localStorage.setItem('plod-lang', lang)` and triggers the game's normal
  re-render path (same mechanism already used for other HUD toggles).

### Wiring into JS-side strings

Strings built directly in JS (not through a `{{ }}` binding) call `t(key)`
at the point they're constructed:

- The `NUKE`/`SURE?` ternary → `s.nukeArm ? t('nukeConfirm') : t('nuke')`.
- The status-line template literal → `` `${t('saved')} ${s.home}/${this.total()} · ${t('needed')} ${this.quota()} · ${t('lost')} ${s.deaths} · ${s.timerStr} ${t('left')}` ``.
- Hub-nav text outside the JSON blob (plain HTML/JS near the end of the
  file: the "Restart" link, "← identity board" link, and any `aria-label`
  attributes on them) → same `t(key)` pattern.

### Wiring into template-bound strings

The `{{ }}` template engine only resolves property paths (confirmed by
inspecting existing bindings like `{{ quota }}`, `{{ t.label }}` inside the
skill-tile `sc-for` loop) — it does not evaluate call expressions, so
`{{ t('play') }}` will not work directly. Every template-bound string
instead gets a thin getter that wraps `t()`, and the template keeps binding
to the getter's name unchanged in shape:

```js
get playLabel() { return t.call(this, 'play'); }
get readyTitle() { return t.call(this, 'readyTitle', { quota: this.quota(), total: this.total() }); }
```

The `SKILLS` array feeding the skill-tile `sc-for` loop gets a parallel
computed getter (e.g. `get skillTiles()`) that maps each entry's existing
`key` (`"climb"`, `"dig"`, etc.) to its translated label via `t()`, so the
loop's binding target changes from `SKILLS` to `skillTiles` but the
per-tile bindings (`t.label`, `t.icon`, `t.hk`, ...) stay the same.

### Language toggle

A small toggle control added to the HUD strip, next to the existing
PAUSE/speed-toggle controls, switching `state.lang` between `en`/`de` on
click — same visual and interaction weight as the existing speed toggle.

### Button width fix

`RESUME`/`RESTART` (currently 180px) and `PLAY AGAIN`/`TRY AGAIN`
(currently 220px) get widened to fit the longer of their EN/DE label pair —
`NEUSTART` (restart) and `NOCHMAL SPIELEN` (play again) are the longest
strings in each pair per the string table below, so both buttons' new
widths are sized against those two words. Exact new pixel widths are
measured during implementation against the actual rendered font.

## Full string table

| key | en | de |
|---|---|---|
| play | PLAY | SPIELEN |
| resume | RESUME | WEITER |
| restart | RESTART | NEUSTART |
| pausedTitle | PAUSED | PAUSE |
| wonTitle | ALL TUCKED IN | ALLE ZU HAUSE |
| playAgain | PLAY AGAIN | NOCHMAL SPIELEN |
| lostTitle | TOO FEW MADE IT | ZU WENIGE GESCHAFFT |
| tryAgain | TRY AGAIN | NOCHMAL |
| climber | CLIMBER | KLETTERER |
| floater | FLOATER | SCHWEBER |
| bomber | BOMBER | BOMBER |
| blocker | BLOCKER | BLOCKER |
| builder | BUILDER | BAUER |
| basher | BASHER | STÜRMER |
| miner | MINER | SCHÜRFER |
| digger | DIGGER | GRÄBER |
| out | OUT | RAUS |
| home | HOME | ZUHAUSE |
| flow | FLOW | FLUSS |
| pause | PAUSE | PAUSE |
| nuke | NUKE | SPRENGEN |
| nukeConfirm | SURE? | SICHER? |
| saved | SAVED | GERETTET |
| needed | NEEDED | BENÖTIGT |
| lost | LOST | VERLOREN |
| left | LEFT | ÜBRIG |
| readyTitle | Guide {quota} of {total} plods home | Führe {quota} von {total} Plods nach Hause |
| readySubtitle | They march out of the hatch and never stop. | Sie marschieren aus der Luke und hören nie auf. |
| readyHint | Pick a skill, click a plod. Paper tears · slate doesn't · foil kills. | Wähle eine Fähigkeit, klicke einen Plod. Papier reißt · Schiefer nicht · Folie tötet. |
| hintSkills | 1–8 skills | 1–8 Fähigkeiten |
| hintPause | SPACE pause | LEERTASTE Pause |
| hintNuke | N nuke ×2 | N Nuke ×2 |
| hintRestart | R restart | R Neustart |
| hotkeys1to8 | hotkeys 1–8 | Tasten 1–8 |
| spacePause | SPACE pause | LEERTASTE Pause |
| nukePress | N nuke (press twice) | N Nuke (2× drücken) |
| rRestart | R restart | R Neustart |
| tweaksHint | accent · total plods · hitboxes in Tweaks | Akzent · Plods gesamt · Hitboxen in Tweaks |
| navRestart | Restart | Neustart |
| navIdentityLink | ← identity board | ← Identitätstafel |

Note: `pause` is identical in both languages (German already uses "Pause"),
included in the table for completeness/consistency rather than special-cased
as untranslated. `hintPause`/`spacePause` and `hintRestart`/`rRestart` are
distinct keys because they're rendered in two different places (ready-screen
hint chip vs. footer bar) with slightly different original English wording
("SPACE pause" vs. the footer's own copy) — kept separate rather than forced
to share a key, so each can be worded naturally for its context. Any
`aria-label` strings on the hub-nav links not enumerated above (exact text
wasn't captured during investigation) get discovered and added to the table
during implementation — same `t(key)` pattern, no design change needed.

## Testing / verification

No automated test suite exists in this repo (single self-contained
`index.html`, no test framework, confirmed during prior work on issue #5).
Verification is manual playtest:

1. Load the game with the manual toggle set to German. Confirm every
   screen — ready, paused, won, lost — HUD strip, skill tiles, footer hint
   bar, and hub-nav all show German text, with no button showing clipped or
   overflowing text.
2. Reload the page. Confirm the language choice persisted (still German).
3. Switch back to English via the toggle. Confirm all strings revert
   correctly and nothing regresses from current behavior.
4. Temporarily remove one key from `STRINGS.de` and confirm `t()` falls
   back to the English value instead of rendering blank/undefined — then
   restore the key.
5. With the manual `localStorage` override cleared, load in a browser
   whose locale is set to German and confirm it defaults to German; load
   with a non-German locale and confirm it defaults to English.

## Out of scope

- `identity.html` (separate visual-identity board page).
- Any language beyond DE/EN.
- Date/number formatting (none is used beyond plain counts and a manually
  built `M:SS` timer string, both locale-independent as-is).
