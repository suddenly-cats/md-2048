# md-2048 — project context

Themed fork of 2048 ("MD 2048", originally ymfa/phd-2048). Plain static site —
vanilla JS + CSS, no build step, no package manager. Deployed via GitHub Pages.

- Live: https://suddenly-cats.github.io/md-2048/
- Repo: https://github.com/suddenly-cats/md-2048

## How tiles work

Tiles are powers of two, but each value is shown as a **named brick** (a career
ladder), not a number. The names are the "tiers".

- **Tier names** live in `js/i18n.js`, in the `captions` array. There are several
  copies: two English variants inside `update_captions()` (one for
  `window.innerWidth < 520`, one for desktop — keep them in sync), plus separate
  Chinese arrays in `use_simplified()` / `use_traditional()` (different,
  shorter PhD theme — usually leave these alone; English is the default).
- **Value → name mapping** is in `js/html_actuator.js` → `val2caption(val)`:
  tile value `2^k` maps to `captions[k-1]`. So index 0 = "Coffee" (2),
  index 10 = "Chief" (2048), and the last index is the highest tier.
  If a value exceeds the array, the raw number is shown (graceful fallback),
  so appending caption entries is always safe.

Current English ladder (index → value → name):
`0:2 Coffee … 10:2048 Chief … 18:524288 GOD, 19:1048576 Bab,
20:2097152 Junior, 21:4194304 "Junior Junior"`.

### Adding a new HIGHEST tier (the easy, non-disruptive case)
1. Append the name to **both** English `captions` arrays in `update_captions()`
   (the new value is automatically 2× the previous highest).
2. That's it for functionality — it will render with the default `tile-super`
   style unless you give it its own color (below).

### Do NOT insert a tier in the middle
The win condition is **hardcoded at value 2048** (`js/game_manager.js`:
`next.value >= 2048` / `merged.value === 2048` set `won`), and progress
messages key off numeric `maxTile` thresholds. Inserting a tier below 2048
silently changes which label sits on the win tile and shifts every higher tier.
Always **append at the top** instead.

## Styling / colors

⚠️ The page loads the **compiled `style/main.css`** — there is **no sass build
step** in the repo, and `style/main.scss` is stale/out of sync (it only defines
tiles up to 2048; the committed CSS has a much richer palette). **Edit
`style/main.css` directly.** Don't trust `main.scss`.

- Per-value tile rules look like `.tile.tile-<value> .tile-inner { ... }`.
- Colors for the high tiers follow a **pastel hue rotation**: orange → green →
  teal → blue → purple → rose-pink (`.tile-super` = `#e0a8b8`) → coral
  (`Junior` `#e0b4a8`) → amber-gold (`Junior Junior` `#e0cca8`). To continue the
  gradient, advance the hue one step in the same pastel family.
- The glow is a `box-shadow`: an outer colored halo + an `inset` white rim.

### Specificity gotcha (important)
`js/html_actuator.js` adds the `tile-super` class to **any** tile with
`value > 2048`. So a high tier gets **both** `tile-<value>` and `tile-super`,
and `.tile.tile-<value> .tile-inner` has the **same specificity** as
`.tile.tile-super .tile-inner`. To make your per-value color win, the new rule
**must appear AFTER the `.tile-super` block** in `main.css` (source order breaks
the tie). The custom Junior / Junior Junior rules are placed right after
`.tile-super`, just before `.karma`.

### Pattern for a new colored + glowing top tier
Mirror the existing `tile-2097152` / `tile-4194304` blocks in `main.css`:
add a `@keyframes <name>-glow` (with `-webkit-`/`-moz-` copies to match the
file's style) and a `.tile.tile-<value> .tile-inner` rule that sets `color`,
`background`, a static `box-shadow` fallback, and the `animation` (also
vendor-prefixed). Include the `@media (max-width: 520px)` font-size override.

## Verifying changes locally

Serve statically and inspect computed styles (do **not** rely on screenshots —
the game runs a continuous `requestAnimationFrame` loop so the preview never
reaches render-idle and screenshots time out):

```
python -m http.server 8723 --directory .
```

Then in the page, read `getComputedStyle('.tile.tile-<value> .tile-inner')` for
`backgroundColor` / `animationName` to confirm the rule applies. To force a
high tile onto the board for visual checks, build a `.tile`/`.tile-inner`
element with classes `tile tile-<value> tile-super` and append it to
`.tile-container` (manipulating `window.game` directly is fragile because the
actuator re-renders async inside rAF).

Quick value→name sanity check (Node):
```
node -e 'const c=[...]; const f=v=>{let i=-1,n=1;while(n<v){n<<=1;i++;}return c[i]??v}; console.log(f(2097152))'
```

## Deployment

GitHub Pages serves from the **`master` branch root** (there is no `gh-pages`
branch despite the upstream Rakefile mentioning one). To deploy: commit to
`master` and `git push origin master`; Pages rebuilds automatically in ~1–2 min.
Verify with a cache-busted fetch of the live asset, e.g.
`curl -s "https://suddenly-cats.github.io/md-2048/js/i18n.js?cb=$(date +%s)"`.
Tell users to hard-refresh (Ctrl+F5) to bypass browser caching of
`i18n.js` / `main.css`.
