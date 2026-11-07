# Moonlit Cloud Sea

An endless, idle flight just above a moonlit sea of clouds, rendered in real time with Three.js. The camera drifts forward along the cloud tops, weaving past cumulus heaps and towering cumulonimbus with wind-dragged anvils. Through breaks in the deck you can see flooded mountains far below: forested ridges and winding lakes catching the moon. A full moon sits low ahead, throwing silver linings and light shafts through the clouds.

Everything is in a single self-contained file, `index.html`. Open it in a current desktop browser with WebGL 2 (Chrome, Edge, Firefox or Safari). Nothing else needs to be served alongside it; the only external request is Three.js r128 from cdnjs.

## Controls

| Input | Action |
| --- | --- |
| Mouse | Gently turns your gaze (heavily damped) |
| Double-click / `F` | Toggle full screen |
| `G` or the corner button | Open the colour-grade panel |
| `L` | Send a lightning flash now |
| `M` | Send a shooting star now |
| `Esc` | Close the panel |

The cursor and the closed panel button fade out after a couple of seconds without mouse movement.

## What's in the scene

**Cloud deck.** A thick stratocumulus layer whose altitude varies across the world. Broad basins sink well below the camera, while banks and ridges swell past eye level to the sides. The flight corridor is kept low so the camera skims over the deck rather than through it. A large-scale coverage field opens breaks of every size, from small ragged gaps to wide openings.

**Cumulus heaps.** Built the way real cumulus grows: a cluster of six rounded thermal bubbles. They're big and spread out at the base, smaller and higher toward the top, and each is nudged downwind. Some come out low and wide, others stack into lopsided turrets. Two scales of rounded lumps on top give the cauliflower texture. (These replaced earlier cylindrical "towers" that read as unnatural.)

**Cumulonimbus.** A few per world tile, mostly mid-sized with occasional giants. Each has a broad, lumpy updraft column that leans slightly with height. The tall ones flatten into an anvil dragged out by upper-level wind, with an overshooting dome punching through the top. They're kept clear of the flight path, so you pass beside and sometimes under the anvils.

**Flooded mountains.** Seen through the breaks, about 85 units below: eroded ranges with sharp ridges, gullies and smooth valley floors, half drowned so the valleys become winding lakes and fjords. Forest covers the gentler slopes below a ragged tree line, in patches with clearings, and gives way to meadow and bare rock on cliffs and peaks, with a dark wet band along the shores. The low moon throws long shadows across the ranges and lays a soft glitter path on the lakes, broader where wind roughens the water. The lakes reflect the sky and the mountains around them. The deck's shadow falls on all of it, so only ground under a break is moonlit.

The terrain never changes and the moon never moves, so nearly all of it is baked at startup into five tiling 1024 × 1024 textures:
- **Height:** the ground plus a tree canopy of crowns grouped in clumps, with forest density and rock grain.
- **Light:** moonlight with soft shadows cast toward the moon, and how much sky each point sees past the surrounding ridges.
- **Colour:** forest, meadow, rock, shore and lake bed.
- **Horizons:** how high the land rises in four directions from every point, which tells a lake when its reflection shows a mountain instead of sky.
- **Lamps:** the warm light the town and road lamps cast on the ground around them, and the lamps themselves as small bright points.

Each frame then only finds where a ray meets the ground (a height-field march through the baked height) and adds the live parts: the deck's shadow, the moon's glitter, and fine crown and rock texture up close. From this high up, ripples on the lakes are far smaller than a pixel, so the water is shaded as a smooth mirror with a little roughness instead of with per-pixel waves. Lightning ground strikes land on the terrain's real surface, read from a CPU copy of the baked heights. Open the page with `?noclouds` to see the ground on its own.

**Town lights.** Small towns of warm, sodium-orange street lights are scattered over the dry, gentle ground near the valley floors and shores. Roughly a third of the sites on a 7 × 7 grid over each world tile get a town, about 18 per tile, though a town that falls mostly on water or cliffs shows only a few lamps. Each town is a loose street grid that thins out toward its edges. Strings of road lamps wind between neighbouring towns, skirting the lakes and steep slopes, with the odd lamp missing. Single lights dot the countryside in between, like isolated farmhouses. Every lamp lights the ground around it with a soft falloff, and the lakes pick up a faint warm glow from nearby towns. All of it is baked into the lamps texture, so drawing it costs one extra texture read per ground pixel. The panel's Ground section sets how bright the lights are, from off to three times the default; it is a single multiplication at render time, needs no rebake, and is saved in the browser.

**Radio masts.** The two highest summits in each world tile carry a thin lattice mast about 1.8 units (roughly 70 m) tall. Each has a red aircraft warning light on top that flashes every two seconds, out of step with the others, and a steady dimmer red light halfway up. A flash faintly reddens the summit around the mast's foot. The summits are picked on the CPU from the read-back heights once the terrain bake finishes. The masts and lights are drawn live in the ground pass with a few simple tests per pixel, so the lights can blink and sit at the real top of the mast. The Town lights slider doesn't affect them.

**Moon and sky.** The moon uses NASA's Lunar Reconnaissance Orbiter albedo map (public domain), embedded in the page. You see the real near side, north up, with the maria and Tycho in their correct places. A layered halo surrounds it. The sky is a deep gradient with a warm glow along the horizon under the moon, and procedural stars that fade near the moon's glare. The stars twinkle irregularly a few times a second, more deeply near the horizon, and reach to within about 2° of it, twice as dense in the lowest part of the sky. A few bright "hero" stars sit low over the horizon ahead, dimmed and reddened by the thick air there. The lowest, nearest the middle, is a steady reddish planet like Mars. The panel's Sky section sets how much the stars twinkle and how close to the horizon they reach; both are saved in the browser.

**Shooting stars.** Now and then, at random (about once every two minutes by default), a meteor streaks across the sky in view. Both ends of its path are picked on screen, so the whole streak is visible, at least 6° up, clear of the moon, heading generally downward and burning out above the horizon. It is a pixel-thin line with a hot blue-green-white head and a warmer trail that fades behind it; it flares up and dies away over 0.3–2 seconds, and about one in twelve is a brighter, longer fireball. Clouds hide it like the stars. The Sky section's Shooting stars slider sets how often they come (off to about every 10 seconds, saved in the browser), and `M` or the panel's button sends one now. With none in the sky they cost nothing.

**Lightning.** Every several seconds one of the cumulonimbus near the camera flashes. About two in three flashes are buried deep in the cloud, one in five is a ground strike, and the rest are in-cloud flashes near the surface.

- **Deep flashes** fire in the core of the tower, well inside the cloud. Their light is scattered through so much cloud that the tower only glows softly from within. They are set to 70% of a ground strike's strength, with slower, gentler pulses and no visible channel.

- **Ground strikes** have a faint stepped leader that works its way down, then one to four return strokes tens of milliseconds apart. Some strokes carry a flickering continuing current. Some strikes leave the cloud base and show only through breaks in the deck. Others leave the side of the tower and arc through clear air before dropping to the ground, which is known as a "bolt from the blue". Before a strike is chosen, ten candidate channels are tested against the baked cloud field, and the one the camera sees best wins.
- **Near-surface in-cloud flashes** fire between charge regions inside the tower as a train of pulses that wander along the channel. On the tall towers with an anvil, some run as "crawlers" just under the anvil base.
- **Cloud lighting.** Each flash also acts as two line lights along its channel inside the cloud ray march. Light from a channel buried in cloud is scattered many times before it escapes, so it fades far more gently than direct moonlight. Three reads of the bake estimate how much cloud lies in between. The result is a hot spot near the channel that spreads through the whole tower. A ground strike also lights the land and water around its strike point and the rainy air under the deck.
- **The bolt** is drawn at full resolution as a jagged, branching channel made by midpoint displacement. Each point of the channel marches the baked cloud field back to the camera, so cloud in front hides it and thin cloud only veils it.
- Storms often flash in quick bursts, so there's a 30% chance that the next flash follows within two seconds, usually from the same tower.
- The panel's Weather section turns lightning off or makes it rarer or more frequent. Its Minimum gap slider (2.5 s to 3 minutes, logarithmic) sets the shortest quiet spell before lightning can flash again, the first flash included; any setting above the far left also turns off the quick bursts, so the gap holds between any two flashes. The `L` key and Flash now button ignore it. These settings are saved in the browser. With "reduce motion" set in the operating system, flashes are softer and single, with no rapid flicker.

**Light shafts.** A screen-space radial pass streaks moonlight past whatever is occluding the moon, so beams fan out when a tower crosses it. It also picks up the brightest silver linings.

## How it renders

Each frame runs five full-screen passes, plus a sixth while a lightning bolt is visible:

1. **Cloud pass** (reduced resolution). Ray-marches the cloud volume per pixel and outputs scattered light, transmittance and a depth estimate.
2. **Ground pass** (same resolution). For pixels that see down through a break, finds and shades the flooded mountains behind whatever cloud is left in front. Every other pixel passes straight through. Keeping it out of the cloud pass keeps that heavy shader lean.
3. **Temporal accumulation.** Blends the new frame with the reprojected history to remove residual noise and stabilise edges.
4. **Light shafts.** Radial blur toward the moon's screen position.
5. **Lightning bolt** (full resolution, only during a flash). Draws the channel as screen-facing strips into its own buffer.
6. **Composite** (full resolution). Draws the sky, stars and moon, combines them with the clouds and shafts, then applies tone mapping and the colour grade.

### The world is one seamless tile

The world is a 320 × 320 unit tile repeated forever in both horizontal directions. Every noise lookup uses a frequency that is a whole number of cycles per tile. The cumulus grid has 13 cells per tile and the cumulonimbus grid has 3. The flight path is a sum of sines with one and two cycles per tile. Because of this the clouds wrap with no seam, and the flight is endless without ever cutting or jumping. At cruising speed the scenery repeats roughly every 4¼ minutes.

The clouds are defined as a signed-distance-like shape field: deck, cumulus and cumulonimbus combined with smooth unions. A tileable 64³ noise texture generated on the CPU at startup provides the detail. Its channels hold Perlin-Worley billows, a Worley detail field and smooth Perlin noise.

### Baked lighting

The moon and the clouds never move, so lighting is computed once on the GPU when the page loads. For every point of the tile, the bake records:

- **Optical depth toward the moon.** This comes from a long march of about 300 units, so tall clouds cast long, soft shadows. Distant parts of a shadow fade, as scattered light would fill them in.
- **Optical depth straight up toward the sky.** This is used for ambient occlusion in basins and under cloud bodies.

- **The cloud shape itself**: the full field (deck, heaps and cumulonimbus with their large billows) as a signed distance.

The bake runs in small chunks across several frames to avoid stalling the GPU. It is rendered into a 2D atlas of slices, read back, and uploaded as a 256 × 96 × 256 3D texture that wraps with the tile. A "Lighting the clouds…" note appears only if baking takes longer than about half a second.

At runtime the cloud shape is read from the bake and one octave of finer live billowing is added on top, instead of rebuilding the whole field per sample. In clear air a sample is a single texture read, and the baked distance also says how far a ray can safely skip. This made the cloud pass roughly 5× cheaper. The bake stores the true field: the above-the-deck shortcut used to speed up skipping is deliberately kept out of it, since baking that shortcut showed up as a phantom sheet of puffs.

Each cloud sample also takes two crisp local samples toward the moon, so edges stay sharp, then reads the bake for everything beyond. Beyond about 70 units only the bake is used, with a smooth blend between 55 and 80 units. Shading combines Beer–Lambert transmittance, a dual-lobe Henyey–Greenstein phase function, a three-octave multiple-scattering approximation, a "powder" term, and height- and occlusion-weighted ambient light. Moonlight shifts from cool silver to warm toward the moon, and haze thins with altitude so distant giants stay visible. Close up the haze is a plain night blue; only cloud beyond about 100 units takes on the sky's colour in its direction (the warm horizon band, the moon's halo), so near clouds don't carry a stripe of horizon glow.

### Sampling built for stability

Most of the development effort went into making the clouds hold still while the camera moves.

- **World-anchored samples.** View rays sample the cloud on planes fixed in the world (constant z) rather than at distances from the camera, so flying forward never slides the sample slices through a cloud. Spacing starts at 0.06 units and doubles at 5, 10, 20, 40, 80 and 160 units out.
- **Blended spacing changes.** Over the far half of each distance band, every other plane fades out and its neighbours take over its share, so the switch to coarser spacing is gradual. This applies to the refined planes near surfaces too.
- **Clean band hand-off.** The band edges move with the camera. When a ray's next sample falls past an edge, it is placed on the coarser band's grid. Otherwise a plane from the finer grid would carry the coarser band's weight and exist only until the edge moved past it, flicking a piece of cloud in and out.
- **Surface refinement.** Near a cloud surface, rays drop to a twice-as-fine set of world-anchored planes, so small puffs and wisps can't slip between samples and pop in and out.
- **Softer distant edges.** Far out, one sample stands for a thick slab, so a sharp density edge turns a tiny shift of a thin wisp into a big jump in opacity. The density ramp at cloud edges widens with distance (`RAMP_K`, at half strength) to keep those changes gradual.
- **Conservative empty-space skipping.** Rays skip through clear air by only a fifth of the baked distance to the nearest cloud. The baked field is not a true distance: near anvils and turrets it changes up to about 6.7× faster than distance, so longer skips could jump straight over a piece of tower.
- **Soft far limit.** Rays march up to 200 units, rising to 320 for rays climbing above the horizon, with the limit blended by ray height. Cloud fades out over the last 45 units before the limit, so planes crossing it as the camera moves don't add or drop whole slabs of distant cloud.
- **No per-frame randomness in the clouds.** With the camera still, consecutive frames are identical.

### Temporal accumulation

The cloud pass writes depth together with transmittance into a 32-bit float target. The accumulation pass uses that depth to reproject each pixel into the previous frame. It clamps the history to the current 3×3 neighbourhood's colour range to prevent ghosting, then blends, leaning more on the new frame where motion is fast. The accumulated image carries over when the render resolution changes, so changes don't reset to noise. Without `EXT_color_buffer_float` the page falls back to unaccumulated rendering.

### Resolution and performance readout

By default the cloud pass renders at 26–80% of the window's size, adjusted automatically from measured frame times with wide hysteresis so changes are rare. The panel's Performance section shows the live cloud resolution, screen resolution and frame rate. A slider pins the cloud resolution anywhere from 15% to 100% of the screen's real pixel width and height; ticking "Adjust automatically" hands control back. The choice is saved in the browser.

## Colour grade

The grade is applied after ACES tone mapping and is driven entirely from the panel. It costs effectively nothing per frame.

- **Light:** exposure, white balance (temperature and tint), light-shaft strength
- **Colour:** saturation, contrast, split toning (shadow and highlight tint colours with strengths), and "night glow" (shadow colour lifted into the blacks)
- **Lens:** vignette, film grain. The grain behaves like film: soft clumps about 1.5 pixels across (scaled with the screen) plus a finer sparkle, mostly neutral with a hint of colour, strongest in the shadows and midtones, and refreshed 24 times a second. A faint separate dither prevents banding even with grain at 0.

**Presets:**

- **Teal Night** (default): blue-green, after the fourth reference image
- **Moonlit Blue:** close to the earlier look
- **Golden Moon:** warm moon over blue clouds
- **Deep Sea:** saturated cyan-teal
- **Silver:** nearly monochrome

Settings are saved in the viewer's browser (`localStorage`).

## Development history

The scene was built iteratively from a brief and four reference photos of moonlit clouds. In order:

1. **First version.** Ray-marched cloud deck with procedural towers, live light marching toward the moon, screen-space light shafts, moon, stars, ACES tone mapping and adaptive resolution.
2. **Breaks in the deck.** The deck was too uniform. A coverage field now opens gaps down to a newly added moonlit ocean.
3. **Darker sky, less glare.** Halo, horizon band and light-shaft veil were all reduced.
4. **Varied deck heights.** Basins and banks were added around a clear flight corridor. A phantom flat layer above the camera turned out to be a bug in the empty-space skipping, which had also caused a hard bright horizon line. Both were fixed.
5. **Shimmer.** Per-frame random sampling made the clouds boil. Temporal accumulation with depth-based reprojection fixed most of it.
6. **Clouds changing shape.** Animated "wind" in the noise was morphing the clouds, and distance-based detail changes made edges reshape as they approached. The clouds were frozen in place and the detail switch pushed far out.
7. **Remaining boiling.** Measured with frame-to-frame difference maps in a headless browser. The cause was sample slices sliding with the camera, fixed with world-anchored sampling planes. Accumulation was also made steadier and preserved across resolution changes.
8. **Baked lighting (Option 2).** The world became a seamless tile and moonlight is baked at startup. Frames got cheaper while sampling got finer.
9. **LOD blending.** Spacing bands and the lighting switch now blend instead of stepping.
10. **Popping.** Fixed with softer ocean glints (the biggest source), surface refinement and more cautious skipping. Large frame-to-frame jumps fell by about 85%.
11. **Real cloud shapes, ocean, moon.**
    - Cumulonimbus with anvils were added and the tall cylinder towers were removed.
    - The ocean texture was rebuilt so it doesn't repeat.
    - The noise-based moon was replaced with the real LRO lunar map.
    - Cumulonimbus heights were varied, with anvils only on the tall ones.
    - The short towers were rebuilt as clusters of thermal bubbles.
12. **Colour grade.** Teal Night became the default, with the preset and control panel added.
13. **Performance readout.** The panel shows the live resolution and frame rate and can pin the cloud resolution.
14. **Baked cloud shape.** Rendering reads a baked copy of the cloud field plus live fine detail, about 5.5× faster per frame in testing, with matching appearance.
15. **Lightning.** Ground strikes and in-cloud flashes from the cumulonimbus, with matching cloud, ground and sky lighting. A held flash adds well under a millisecond per frame in testing.
16. **Edge popping.** Small pieces at the sides and tops of clouds, especially the cumulonimbus, blinked in and out as the camera moved. Five sources were traced to individual pixels:
    - the hard end of the march, now a soft far limit;
    - skips that leapt over cloud because the baked field understates distance near steep shapes, now shorter;
    - samples that flicked in and out where the distance bands change spacing, now a clean hand-off;
    - thin distant wisps with edges sharper than the sample spacing, now softened with distance;
    - stars cut off at the faces of their hash cells, now placed on the sky sphere and checked across neighbouring cells.

    Measured at 61% cloud resolution over 4 positions and 48 frames each, cloud jumps larger than 32 brightness levels fell from 327 to 4, and jumps larger than 64 levels from 112 to none. The frame costs about 15% more GPU time. Most of that is the march now sampling planes it used to skip by mistake. Sub-pixel jitter was also tried and made no visible difference, so it was left out.
17. **Near-cloud fog.** Fog took its colour from the sky in each pixel's direction, so close clouds crossing the horizon on screen wore a band of horizon glow. Close-range fog is now a plain night blue that blends into the sky's colour between 100 and 250 units.
18. **Flooded mountains.** The ocean's waves read as a curly texture rather than water, so the ground became flooded mountains with forests and lakes, baked into tiling textures (height with canopy, moonlight and shadows, sky visibility, colour, horizons). It is drawn in its own pass, and costs slightly less than the old ocean did: about 49 ms against 54 ms per frame in a GPU-timed comparison at 61% cloud resolution in a hidden browser pane, where absolute times run high. The terrain bake runs after the cloud bake at startup, in strips across several frames.
19. **Stars near the horizon.** The star field had faded out well above the horizon and pulsed too slowly to read as twinkling. Stars now reach almost to the horizon, a few bright hero stars and a Mars-like planet sit low ahead, and every star flickers a few times a second. The panel has Twinkle and Stars reach down to controls.
20. **Town lights.** Warm street lights in small towns, lamp-lit roads between them and scattered farmhouse lights, all baked into a fifth terrain texture that also holds the light they cast on the ground. The panel's Ground section sets their brightness at no cost.
21. **Radio masts.** Thin masts on a few summits with flashing red aircraft warning lights, drawn live in the ground pass.
22. **Film grain.** The grain slider barely did anything: it was single-pixel noise of at most ±1.5%. It is now film-style grain, about six times stronger at the same setting.
23. **Shooting stars.** Occasional meteors across the part of the sky in view, with a rate slider and an `M` key to send one.
24. **Lightning minimum gap.** A Weather slider for the shortest time between flashes, which also turns off quick bursts when set above its default.

## Known limitations and ideas

- **Visible repeat.** The scenery repeats every ~4¼ minutes. Streaming the bake (baking new sky ahead of the camera and discarding what's behind) would make it never repeat.
- **Startup bake time.** The bake takes a second or two on a typical GPU, longer on integrated graphics.
- **Slower GPUs.** Integrated graphics will run at reduced cloud resolution, so the image is softer.
- **Distant anvils** can occasionally read as mushroom caps. Stretching them further downwind and thinning their edges would help.
- **Clouds in the shadow** of a large cumulonimbus can look flat. Skylight relief was strengthened, but there's room for more.
- **Possible next step:** temporal upsampling (rendering a quarter of the cloud pixels each frame and rebuilding the rest from recent frames) could add another 3–4× on top of the shape bake.

## Files

- `index.html`: the complete scene (Three.js r128 from cdnjs; everything else inline)
- `assets/moon-lroc-512.jpg`: the lunar albedo map embedded in `index.html`, kept here for reference

## Credits

- Lunar map: NASA / GSFC Scientific Visualization Studio, CGI Moon Kit (LRO LROC data). Public domain.
- Three.js: MIT licence.
