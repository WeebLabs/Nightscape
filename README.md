# Moonlit Cloud Sea

An endless, idle flight just above a moonlit sea of clouds, rendered in real time with Three.js. The camera drifts forward along the cloud tops, weaving past cumulus heaps and towering cumulonimbus with wind-dragged anvils. Through breaks in the deck you can see the night ocean far below. A full moon sits low ahead, throwing silver linings and light shafts through the clouds.

Everything is in a single self-contained file, `index.html`. Open it in a current desktop browser with WebGL 2 (Chrome, Edge, Firefox or Safari). Nothing else needs to be served alongside it; the only external request is Three.js r128 from cdnjs.

## Controls

| Input | Action |
| --- | --- |
| Mouse | Gently turns your gaze (heavily damped) |
| Double-click / `F` | Toggle full screen |
| `G` or the corner button | Open the colour-grade panel |
| `L` | Send a lightning flash now |
| `Esc` | Close the panel |

The cursor and the closed panel button fade out after a couple of seconds without mouse movement.

## What's in the scene

**Cloud deck.** A thick stratocumulus layer whose altitude varies across the world. Broad basins sink well below the camera, while banks and ridges swell past eye level to the sides. The flight corridor is kept low so the camera skims over the deck rather than through it. A large-scale coverage field opens breaks of every size, from small ragged gaps to wide openings.

**Cumulus heaps.** Built the way real cumulus grows: a cluster of six rounded thermal bubbles. They're big and spread out at the base, smaller and higher toward the top, and each is nudged downwind. Some come out low and wide, others stack into lopsided turrets. Two scales of rounded lumps on top give the cauliflower texture. (These replaced earlier cylindrical "towers" that read as unnatural.)

**Cumulonimbus.** A few per world tile, mostly mid-sized with occasional giants. Each has a broad, lumpy updraft column that leans slightly with height. The tall ones flatten into an anvil dragged out by upper-level wind, with an overshooting dome punching through the top. They're kept clear of the flight path, so you pass beside and sometimes under the anvils.

**Ocean.** Seen through the breaks, about 85 units below. The waves are four octaves at unrelated scales and angles, plus a slow domain warp and calm/rough "wind slick" patches, so nothing repeats visibly. The moon lays a soft glittering path on the water. That path is dimmed wherever the deck blocks the moon's line to that patch of sea.

**Moon and sky.** The moon uses NASA's Lunar Reconnaissance Orbiter albedo map (public domain), embedded in the page. You see the real near side, north up, with the maria and Tycho in their correct places. A layered halo surrounds it. The sky is a deep gradient with a warm glow along the horizon under the moon, and procedural twinkling stars that fade near the moon's glare and the horizon.

**Lightning.** Every several seconds one of the cumulonimbus near the camera flashes. About a third of the flashes are ground strikes and the rest stay in the cloud.

- **Ground strikes** have a faint stepped leader that works its way down, then one to four return strokes tens of milliseconds apart. Some strokes carry a flickering continuing current. Some strikes leave the cloud base and show only through breaks in the deck. Others leave the side of the tower and arc through clear air before dropping to the sea, which is known as a "bolt from the blue". Before a strike is chosen, ten candidate channels are tested against the baked cloud field, and the one the camera sees best wins.
- **In-cloud flashes** fire between charge regions inside the tower as a train of pulses that wander along the channel. On the tall towers with an anvil, some run as "crawlers" just under the anvil base.
- **Cloud lighting.** Each flash also acts as two line lights along its channel inside the cloud ray march. Light from a channel buried in cloud is scattered many times before it escapes, so it fades far more gently than direct moonlight. Three reads of the bake estimate how much cloud lies in between. The result is a hot spot near the channel that spreads through the whole tower. A ground strike also lights the sea around its strike point and the rainy air under the deck.
- **The bolt** is drawn at full resolution as a jagged, branching channel made by midpoint displacement. Each point of the channel marches the baked cloud field back to the camera, so cloud in front hides it and thin cloud only veils it.
- Storms often flash in quick bursts, so there's a 30% chance that the next flash follows within two seconds, usually from the same tower.
- The panel's Weather section turns lightning off or makes it rarer or more frequent. These settings are saved in the browser. With "reduce motion" set in the operating system, flashes are softer and single, with no rapid flicker.

**Light shafts.** A screen-space radial pass streaks moonlight past whatever is occluding the moon, so beams fan out when a tower crosses it. It also picks up the brightest silver linings.

## How it renders

Each frame runs four full-screen passes, plus a fifth while a lightning bolt is visible:

1. **Cloud pass** (reduced resolution). Ray-marches the cloud volume per pixel and outputs scattered light, transmittance and a depth estimate.
2. **Temporal accumulation.** Blends the new frame with the reprojected history to remove residual noise and stabilise edges.
3. **Light shafts.** Radial blur toward the moon's screen position.
4. **Lightning bolt** (full resolution, only during a flash). Draws the channel as screen-facing strips into its own buffer.
5. **Composite** (full resolution). Draws the sky, stars and moon, combines them with the clouds and shafts, then applies tone mapping and the colour grade.

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

Each cloud sample also takes two crisp local samples toward the moon, so edges stay sharp, then reads the bake for everything beyond. Beyond about 70 units only the bake is used, with a smooth blend between 55 and 80 units. Shading combines Beer–Lambert transmittance, a dual-lobe Henyey–Greenstein phase function, a three-octave multiple-scattering approximation, a "powder" term, and height- and occlusion-weighted ambient light. Moonlight shifts from cool silver to warm toward the moon, and haze thins with altitude so distant giants stay visible.

### Sampling built for stability

Most of the development effort went into making the clouds hold still while the camera moves.

- **World-anchored samples.** View rays sample the cloud on planes fixed in the world (constant z) rather than at distances from the camera, so flying forward never slides the sample slices through a cloud. Spacing starts at 0.06 units and doubles at 5, 10, 20, 40, 80 and 160 units out.
- **Blended spacing changes.** Over the far half of each distance band, every other plane fades out and its neighbours take over its share, so the switch to coarser spacing is gradual.
- **Surface refinement.** Near a cloud surface, rays drop to a twice-as-fine set of world-anchored planes, so small puffs and wisps can't slip between samples and pop in and out.
- **Conservative empty-space skipping.** Rays skip quickly through clear air but cautiously enough not to leap over thin cloud edges.
- **No per-frame randomness in the clouds.** With the camera still, consecutive frames are identical.

### Temporal accumulation

The cloud pass writes depth together with transmittance into a 32-bit float target. The accumulation pass uses that depth to reproject each pixel into the previous frame. It clamps the history to the current 3×3 neighbourhood's colour range to prevent ghosting, then blends, leaning more on the new frame where motion is fast. The accumulated image carries over when the render resolution changes, so changes don't reset to noise. Without `EXT_color_buffer_float` the page falls back to unaccumulated rendering.

### Resolution and performance readout

By default the cloud pass renders at 26–80% of the window's size, adjusted automatically from measured frame times with wide hysteresis so changes are rare. The panel's Performance section shows the live cloud resolution, screen resolution and frame rate. A slider pins the cloud resolution anywhere from 15% to 100% of the screen's real pixel width and height; ticking "Adjust automatically" hands control back. The choice is saved in the browser.

## Colour grade

The grade is applied after ACES tone mapping and is driven entirely from the panel. It costs effectively nothing per frame.

- **Light:** exposure, white balance (temperature and tint), light-shaft strength
- **Colour:** saturation, contrast, split toning (shadow and highlight tint colours with strengths), and "night glow" (shadow colour lifted into the blacks)
- **Lens:** vignette, film grain

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
15. **Lightning.** Ground strikes and in-cloud flashes from the cumulonimbus, with matching cloud, sea and sky lighting. A held flash adds well under a millisecond per frame in testing.

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
