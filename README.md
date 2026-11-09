# ASCII fireworks

A single page. Click the sky and a rocket climbs to that point, coasts to a stop, and bursts into ASCII sparks. Everything lives in [`index.html`](index.html): the markup, the panel, and the animation. There is no build step and no dependency.

Open the file in a browser, or serve the folder:

```
python -m http.server
```

Then open `http://127.0.0.1:8000/`.

## What the page is

The picture is a grid of monospace cells, not a bitmap of particles. Each cell holds at most one character. Sparks are simulated in continuous coordinates, then snapped onto that grid every frame. The canvas is cleared and rebuilt from the live rockets and sparks, so nothing is left behind from the previous frame.

The panel in the top right does not launch a firework. Its `pointerdown` handler stops the event before it reaches the canvas.

## Skeleton

- `#sky` is the canvas, sized to the viewport.
- `#panel` holds the sliders and the two buttons.
- The script is one classic script. `layout()` runs once at the bottom, then `requestAnimationFrame(frame)` starts the loop.

The canvas context is created with `alpha: false`. Its transform is the device pixel ratio, capped at 2, so all of the drawing code can think in CSS pixels.

## The grid

`layout()` runs on load and on resize.

1. `cols` is the Columns slider, rounded.
2. `cellW = viewportWidth / cols`.
3. `cellH = cellW * aspect`, then `rows = round(viewportHeight / cellH)`.
4. `cellH` is stretched once more so the rows fill the height exactly.

Cells are taller than they are wide. The Height slider is that aspect ratio, not a row count. The label next to Columns shows `cols × rows` because the row count follows the window.

Three arrays cover every cell:

| Array | Holds |
| --- | --- |
| `energy` | `0` if the cell is empty, otherwise a rank |
| `glyphs` | the character to draw |
| `colors` | the color packed as `R << 16 \| G << 8 \| B` |

`lit` is a compact list of the indexes written this frame. Drawing walks `lit`, not the whole grid.

## One frame

`frame(now)`:

1. `dt` is the time since the last frame, in seconds, capped at `0.05` so a hitch cannot teleport a spark.
2. `litCount` is reset and `energy` is filled with zeros. Glyph and color arrays are left dirty; an empty energy cell is never read.
3. `step(dt)` moves rockets and sparks and stamps characters back onto the grid.
4. `draw()` paints the sky, then the lit cells.

## A click

`launch(x, y)` picks a color scheme, then either bursts immediately or starts a rocket.

`pickScheme()` returns a rainbow about 14% of the time. Otherwise it picks one of four neighboring pairs. Each pair is two HSL colors a short step apart on the wheel:

| Pair | Hues |
| --- | --- |
| Red and orange | 2 and 28 |
| Yellow and lime | 50 and 78 |
| Green and teal | 126 and 156 |
| Blue and violet | 208 and 246 |

A paired firework gives each arm one of those two colors (`slot % 2`). A rainbow arm takes its hue from its angle: `((angle + π) / 2π) * 360`, at saturation 88 and lightness 60. The rocket itself, when it is a rainbow, cycles hue with age and snaps that hue to the nearest 2 degrees so the letter cache stays bounded.

If the click is within `5 * cellH` of the bottom, there is no room to climb, and `explode` runs at the cursor. Otherwise a rocket is born at the bottom (`y = viewportHeight - cellH * 0.5`) and aimed at the click. Its duration is `room / (viewportHeight * 1.15)`, clamped to `0.28…0.78` seconds.

## The climb

The rocket is not integrated. Its position is a function of age.

```
t = min(1, age / duration)
ease = 1 - (1 - t)³
y = startY + (targetY - startY) * ease
```

The cubic ease starts fast and arrives at the click with zero speed.

The x position is the click, plus a wiggle. Two sines (about 1.35 and 2.6 cycles across the flight) are mixed 70/30 and scaled by `5.5 * cellW`. An envelope fades the wiggle in over the first 14% of the flight and back out over the last 20%, so the rocket leaves the bottom and arrives on the click.

A trail point is stored when the head has moved more than `0.7 * cellH` vertically or `0.75 * cellW` horizontally. The visible length is `round(84 * (1 - t)²)`. That is 84 points at launch and 0 at the peak. `trailStart` advances through the array instead of shifting it; the consumed prefix is spliced off once it is longer than 48.

At `t >= 1` the phase becomes `"gone"`, the trail is cleared, and the rocket waits `0.16` seconds with nothing on screen. Then it explodes at the click and is removed.

The climb is stamped with `stamp`. The first writer of a cell wins. Later trail points that land on the same cell are ignored. The head uses energy `1`; older points use `0.28 + 0.45 * (index / count)`, which picks a thinner character band.

## The burst

`burstReach()` is `max(0.35, rows / 32) * size`. Speed, gravity, and travel all scale with it, so the Size slider changes the shell without a separate radius.

`explode` builds two groups:

- **Arms.** `max(4, round(arms * max(0.75, scale)))` sparks, spaced evenly around a full circle. Asymmetry jitters each angle by up to half a step and stretches its speed. The launch speed is `(38 + random * 14) * stretch * cellH * reach` pixels per second. Life is about `2.2…2.9` seconds.
- **Fuzz.** About 42% as many sparks, any direction, at a much lower speed (`4…15` times `cellH * reach`) and a shorter life (`0.75…1.3` seconds). They stay in the core.

There is no second explosion. One click produces one burst.

### Depth, without growing past the flat radius

Each spark has a `zDir`. Screen speed is the flat speed multiplied by `sqrt(1 - z²)`, and by an extra `1 / (1 + 0.9z)` only when `z` is positive (going away). A spark in the screen plane (`z = 0`) travels exactly as far as the old flat burst. Nothing is magnified past that rim.

Arms repeat in threes:

| `i % 3` | `z` | What you see |
| --- | --- | --- |
| 0 | `0` | The round outline, full distance |
| 2 | `-0.6` | Toward the viewer, shorter, drawn on top |
| 1 | `0.6` | Away from the viewer, shorter still, thin dot glyphs |

Fuzz alternates `z = 0.45` and `z = -0.45`.

`glyphChar` uses the faintest band (`BANDS[0]`) when `z > 0.2`. Everything else picks from the heavier bands.

## Spark motion

A spark stores its launch velocity in `ovx` and `ovy` and copies it into `vx` and `vy` on the first step. There is no windup.

Each step, in order:

1. **Drag.** `drag = 2.4 + speed * 0.0032`, then both velocity components are multiplied by `exp(-drag * dt)`. The shell leaves fast and is nearly stopped by about 0.8 seconds.
2. **Gravity, late.** `g = 16 * cellH * burstReach * gravity * spark.grav`. It is added to `vy` only after the current speed has fallen below 62% of the launch speed. The shell stays round while it is exploding, then falls. `spark.grav` is `0.72…1` for arms and `0.35…0.65` for fuzz.
3. **Wobble.** A small random shove, damped by `exp(-2.4 * dt)`, is applied perpendicular to the velocity. Noise scales it.
4. **Integrate.** `x += vx * dt`, `y += vy * dt`.

The fall starts only when the spark is actually coasting: speed under 42% of launch, `vy` downward by more than half a cell per second, and age past `0.34` seconds. It also starts unconditionally after half the spark's life. An arm that is fired downward must not enter the fall on the first frame, or the bottom half of the shell disappears and the burst looks like an umbrella.

Glyphs are laid along the segment since the last sample, at a spacing of `min(cellW, cellH) * 0.42`. Each glyph is one character, jittered by a fraction of a cell. Until the spark is falling, `dieAt` is infinity. `beginFall` then assigns a die time: newer glyphs (higher index along the arm) tend to last longer, and the Fade slider stretches or shortens that window through `fadeMul = 1.35 - fade`.

A spark is removed when its age passes its life, when it is falling and its glyphs are gone, or when it has left the bottom of the screen. If more than 4200 sparks exist, the oldest are dropped.

## One character per cell

Every living glyph is stamped every frame, including glyphs that land on a cell another spark already claimed. `stampChar` resolves that:

- Rank is `1 - zDir`. Toward the viewer is about `1.6`, the screen plane is `1`, and away is about `0.4`.
- An empty cell takes the glyph.
- A cell whose energy is already greater than or equal to the new rank keeps what it has.
- A higher rank replaces the character and the color. The cell is not added to `lit` a second time.

`draw` paints each lit cell once. Overlapping particles are simulated and tested, and they are not stacked on top of each other.

`BANDS` runs from sparse punctuation up to dense characters (`@`, `#`, `%`). The rocket's energy picks a band. Sparks pick a band from depth instead.

## Drawing

Letters are rasterized once and reused.

`makeSprite` draws a character into a scratch canvas at the current font (`floor(cellH * 0.92)` px Consolas), then copies that bitmap into a 1024×1024 atlas. `paintGlyph` looks the sprite up by packed color and character. A miss rasterizes; a hit is a `drawImage` of that atlas region.

`draw`:

1. Fill the sky white or black.
2. If anything is lit, clear an offscreen color canvas and paint every lit cell into it from the atlas.
3. If the glow is on **and** the sky is white, copy that color canvas, punch it to a black silhouette with `source-in`, and draw the silhouette twice with `shadowBlur = max(16, cellH * 4)`. The sharp color canvas is then drawn on top. Two blurs match the weight of the old per-character shadow. The blur is of the whole burst, not of each letter.
4. If the glow is off, or the sky is black, skip step 3. A black shadow on a black sky adds nothing, so skipping it does not change the picture.
5. If the sky is idle, draw the word `click` in the middle.

The glow button starts off. Its label is the action it will take: **Glow on** while the glow is off.

The atlas is thrown away once it spans more than three pages, and cache entries that have not been used for 64 frames are dropped. Resizing the window rebuilds the atlas because the font size changed.

Removing a rocket, spark, or glyph uses `dropAt`: swap the last item into the hole and pop. In the reverse loops, the index is revisited only when that swap actually put a new item there.

## Controls

| Slider | Default | What it changes |
| --- | --- | --- |
| Columns | 302 | Cell width, and therefore how many rows fit |
| Height | 1.70 | `cellH / cellW` before the rows are fit to the window |
| Size | 0.40× | `burstReach`, so speed, gravity, and travel together |
| Arms | 54 | Ray count. Fuzz is 42% of that |
| Gravity | 1.35× | The downward acceleration after the shell slows |
| Asymmetry | 75% | Angle jitter and speed stretch |
| Noise | 60% | Wobble and the jitter when a glyph is placed |
| Fade | 35% | How long falling glyphs survive |

**White background** / **Black background** flips `lightSky`, the page background, and the clear color. A firework keeps the colors it was born with.

Changing Columns or Height calls `layout()` and rebuilds the grid. The other sliders only update `controls` and the readout. Bursts already in the air pick up gravity, noise, and fade on the next frame. Arm count and size apply to the next explosion.

## Build it in this order

1. **A black fullscreen canvas and a `requestAnimationFrame` loop** that clears and caps `dt`.
2. **The grid.** Derive `cellW`, `cellH`, `cols`, and `rows` as above. Keep `energy`, `glyphs`, and `colors`.
3. **`stamp`.** Map a point to a cell, refuse anything outside the grid, and keep the first writer.
4. **The climb.** Parametric ease, wiggle, a trail whose length is `84 * (1 - t)²`, a `0.16` s empty hold, then explode. Clicks near the bottom skip the climb.
5. **Arms around a full circle**, one character wide, with fuzz in the middle. Speed them with isotropic drag so they start fast and stop.
6. **Delay gravity** until speed is under 62% of launch, and delay the fall until the spark is coasting or half its life has passed. This is what keeps the shell a circle instead of an umbrella or a tall oval. Do not divide horizontal speed by the cell aspect; the circle is in screen pixels.
7. **Depth.** Multiply screen speed by `sqrt(1 - z²)`, and foreshorten only positive `z`. Leave every third arm at `z = 0` so the rim stays the flat radius.
8. **Colors.** Four neighboring pairs, plus the occasional rainbow. One color per arm.
9. **Overlap.** Give `stampChar` a rank of `1 - zDir` so a nearer spark replaces a farther one, and record the cell in `lit` only the first time it is claimed.
10. **The atlas and the single glow blur**, and skip the glow on a black sky.

## Where to change the look

- Trail length is the `84` in `step`.
- The empty beat before the burst is `r.hold = 0.16`.
- Drag is `2.4 + speed * 0.0032`. Raising the constant makes the shell stop sooner.
- The gravity gate is `launchSpeed * 0.62`. Lower it and the shell starts falling while it is still round and fast.
- Depth steps are `0.6` and `-0.6` in `explode`. The rim stays at `0`.
- Glow radius is `max(16, cellH * 4)` in `draw`, and the silhouette is drawn twice.
- Pairs live in `COLOR_PAIRS` as `[hue, saturation, lightness]`.
