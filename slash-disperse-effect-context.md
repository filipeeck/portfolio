# Slash-Disperse Effect — Context Brief

## What it does
When the cursor moves over the gradient text "interfaces that feel obvious" in the welcome H1, it acts like a knife: the text gets sliced where the cursor passes (parts disappear), particles fly off in the cursor's direction, then the text heals/reforms. Faster cursor → bigger slash, more particles, farther travel.

## Tech stack
- **Plain HTML/CSS/JS** in a single `index.html` (portfolio site).
- Target element: `.welcome h1 .accent-grad` (inline `<span>` with a CSS gradient via `background-clip: text`). The span **can wrap to multiple lines**.
- Desktop only — disabled via `window.matchMedia('(hover: none), (pointer: coarse)').matches`.

## Architecture (3 pieces)
1. **SVG mask** applied to the gradient span via CSS `mask-image: url(#id)`. White = visible, black circles = "cuts" (invisible). Circles shrink over time so the text reforms.
2. **Canvas overlay** (fixed, full-viewport, `z-index: 8500`) for particles. Single canvas, one draw call per particle per frame.
3. **rAF loop** that updates cuts (shrinking) and particles (move + fade) until both are empty.

## Key gotcha: SVG mask units on multi-line inline elements
This bit three times. The fix is two-part:

**1. Make the target span `display: inline-block`.** Inline spans that wrap to multiple lines get sliced per line box, and SVG masks on sliced inline elements don't share one continuous coordinate space. `inline-block` gives the span a single atomic box. The text inside can still wrap (`white-space: normal; max-width: 100%`) without affecting the mask.

**2. Use `userSpaceOnUse` for both `maskUnits` AND `maskContentUnits`**:
```js
mask.setAttribute('maskUnits', 'userSpaceOnUse');
mask.setAttribute('maskContentUnits', 'userSpaceOnUse');
mask.setAttribute('x', '-99999');
mask.setAttribute('y', '-99999');
mask.setAttribute('width', '999999');
mask.setAttribute('height', '999999');
```
Cuts at `(cursorX - bboxLeft, cursorY - bboxTop)` now align with the cursor.

- `objectBoundingBox` content gave broken per-line slicing.
- Base rect must be huge (`x=-99999, width=999999`) to cover everywhere not cut.

## Mouse → effect mapping
On `mousemove` inside the span, compute `dx, dy, dist, speed` from last mouse event. Then:
```
sc = min(2.6, speed)                       // px/ms, clamped
slashRadius   = 9  + sc * 14               // 9–45px
particleSpeed = 1.2 + sc * 4.4             // 1.2–12.6 px/frame
particleCount = floor(6 + sc * 26)         // 6–73
cutDuration   = 520 + sc * 480             // 520–1770ms
```
**Cuts:** sample positions along the path between last and current cursor, every ~12px, and add a black circle to the mask at each.
**Particles:** spawn `particleCount` particles randomly along the path within `slashRadius`. Each particle's velocity = cursor direction (`atan2(dy, dx)`) ± 0.65 rad spread, magnitude scales with cursor speed.

## Cut lifecycle
Each cut holds full radius for the first 18% of its `duration`, then eases closed with `1 - (1-u)^2.5`. Removed when `r ≤ 0`.

## Particle lifecycle
```js
p.vx *= 0.985; p.vy *= 0.985; p.vy += 0.022;  // air drag + light gravity
p.life -= p.decay;                             // decay random 0.008–0.026
```
Drawn as 1–2.5px squares with `rgba(r,g,b, life*1.5)`. Removed when `life ≤ 0`.

## Particle colors
Sampled from the gradient stops by horizontal position:
```
0    #ff7e87  → 1/6  #ffb88c  → 2/6  #b48aff  → 3/6  #5e8df4
→ 4/6  #34d399  → 5/6  #b48aff  → 1    #ff7e87
```
Linear interpolation between adjacent stops based on `(x - bboxLeft) / bboxWidth`.

## Element setup
- The SVG mask wrapper is `position: absolute; left: -9999px` so it's offscreen but the mask is referenceable.
- The `.accent-grad` span needs `position: relative; cursor: default` and the mask applied via inline style on init:
  ```js
  grad.style.maskImage = 'url(#'+maskId+')';
  grad.style.webkitMaskImage = 'url(#'+maskId+')';
  ```

## Things to preserve / not break
- The CSS gradient animation on the span (`gradient-drift` keyframes, 8s loop) — leave it alone.
- The custom cursor JS (separate from this).
- `prefers-reduced-motion` — the effect should probably skip if reduced motion is requested (currently doesn't; can add).

## Failure modes to watch
- If cuts misalign with cursor: check `maskContentUnits` is `userSpaceOnUse`, not `objectBoundingBox`.
- If the whole text vanishes: base rect is too small. Make sure it's `width="999999" height="999999"` with negative `x/y`.
- If nothing renders: SVG must be in the DOM (appended to body) before the `mask-image` URL resolves.

That's everything. The implementation is ~150 lines of JS + a tiny bit of CSS. The whole effect lives in one IIFE you can drop into a portfolio site.
