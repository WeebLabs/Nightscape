# Investigation: Nightscape freezes while loading in Chrome on Windows

This is a briefing for a Claude session on a Windows machine. It was written by a session on a Mac that could not reproduce the problem. Read it all before changing anything. Report what you find to the user, and add your findings to the end of this file.

## The problem, in the user's words

The user opens the live demo in Chrome on Windows. The loading bar starts filling, gets to about two-thirds, and freezes. It never moves again by itself. While it is frozen, clicking anything else in the browser does not respond. If the user clicks another tab, that tab eventually opens. When they switch back to the demo tab, the loading bar finishes and the demo then works normally.

The important point: **this is not slowness.** Left alone, it never finishes. Switching away from the tab and back is what unsticks it. The problem predates all recent work, so it is not caused by the title screen or the loading bar, although the bar now shows where it stops.

The freeze does not happen in Chrome on an Apple Silicon Mac (ANGLE on Metal), where the whole startup takes about 1.5 seconds of work.

## The project

- **Live demo:** <https://weeblabs.github.io/Nightscape/>. GitHub Pages serves it from the `main` branch, so pushing `main` updates it within a minute or two.
- **Everything is in `index.html`**: one page with WebGL 2 shaders and Three.js r128 from cdnjs. There is no build step.
- **To run it locally**, serve the repository folder over HTTP and open the page:

  ```
  py -m http.server 8765
  ```

  Then open <http://localhost:8765/?diag>.

## How startup works, and where two-thirds falls

The main script (`function nightscape()`) starts after the page's first paint, so the title can show during setup. Setup itself takes about 0.4 s on the main thread. After that, the render loop (`function loop`) does one startup step per animation frame until `baked` is true:

1. **Cloud bake: 48 chunks** (`bakeStep`). `BAKE_FRAG` renders into `bakeRT`, a 2048 × 3072 RGBA8 render target used as an atlas of 96 slices of 256 × 256. Each chunk is a scissored draw of 512 × 256 pixels, now cut into strips whose height adapts to the frame interval. The first chunk compiles the bake shader. Each pixel evaluates the cloud shape field about 240 times: 150 steps toward the moon, 14 up and 5 × 16 down.
2. **Cloud bake read-back** (end of `bakeStep`): `renderer.readRenderTargetPixels` reads the whole 25 MB atlas back to the CPU. JavaScript repacks it into `bakeTex.image.data`, a 256 × 256 × 96 `Data3DTexture`, sets `needsUpdate`, and disposes `bakeRT`. The 3D texture is uploaded when it is first used, which is the first scene frame.
3. **Terrain bake: 26 jobs** (`terrainStep`, list `TER_JOBS`). These are scissored draws into 1024 × 1024 targets:
   - Jobs 1–4: height (`TER_H_FRAG`; job 1 compiles it).
   - Jobs 5–12: moonlight (`TER_L_FRAG` mode 0; job 5 compiles it).
   - Jobs 13–14: colour (`TER_L_FRAG` mode 1).
   - Jobs 15–18: horizons (`TER_L_FRAG` mode 2).
   - Jobs 19–26: town lights (`TER_T_FRAG`; job 19 compiles it).
4. **Terrain read-back**: the 1024 × 1024 height target (half float where available) is read back for lightning and the radio masts.
5. **First scene frames**: the first frame compiles the cloud, ground, temporal-accumulation, light-shaft and composite shaders, and uploads the 3D bake texture.

The loading bar shows `(cloud chunks done + terrain jobs done) / (48 + 26 + 1)`. **Two-thirds is 48 of 75**, the end of the cloud bake. So the freeze is almost certainly in one of these:

- the 25 MB read-back of the cloud bake;
- terrain job 1, which compiles `TER_H_FRAG`;
- terrain job 5, at about 69%, which compiles `TER_L_FRAG`;
- or the GPU still working through queued cloud-bake strips, which the read-back must wait for.

## What has been tried, and did not fix it

Commit `8f7dc25`, which is live:

- **Texture reads at an explicit level.** Every read of the 3D noise and the bake inside the shaders' loops now uses `textureLod(…, 0.0)`, as does the light-shaft loop's read of the cloud image. This keeps Microsoft's HLSL compiler (FXC), which Chrome's ANGLE uses on Direct3D, from having to unroll those loops. Neither texture has mipmaps, so the output is identical.
- **Adaptive bake strips.** Each cloud-bake chunk is drawn in strips that halve while frames come more than 45 ms apart and double while they come less than 24 ms apart.

Neither fixed the freeze. The adaptive strips probably cannot help: while loading, nothing is drawn to the canvas, so the browser never waits for the GPU between frames. The whole bake can therefore be queued within a second, and the read-back then waits for all of it.

## Leading suspects, and how to test each

1. **Queued GPU work that is never submitted until the tab's visibility changes.** This fits "never finishes, but switching tabs unsticks it" best. The page renders only to offscreen targets during loading and never presents a frame. If Chrome or ANGLE on Windows holds that work back until something forces a flush, the read-back waits forever. Switching tabs forces it.
   - *Tests:* call `gl.flush()` after each bake strip. Or draw something trivial to the canvas each loading frame (for example, clear the default framebuffer) so frames are presented. Or wait on a fence before the read-back:
     ```js
     gl.fenceSync(gl.SYNC_GPU_COMMANDS_COMPLETE, 0)
     gl.getSyncParameter(sync, gl.SYNC_STATUS)
     ```
     Then check whether the freeze goes away.
2. **The 25 MB synchronous read-back itself** stalling or deadlocking on Direct3D 11.
   - *Tests:* read back in several smaller pieces across frames. Or read asynchronously into a `PIXEL_PACK_BUFFER` guarded by a fence. Or avoid the read-back entirely by rendering the bake straight into the layers of the 3D texture, using `framebufferTextureLayer` in WebGL 2.
   - The cloud bake size can be reduced for a test by setting `window.__cloudDebug = { bakeN: 128 }` before the page's scripts run, for example in a local copy of `index.html`. Note that any `__cloudDebug` value also turns off the title screen and the automatic lightning, aircraft and shooting stars.
3. **Shader compile on Direct3D** of `TER_H_FRAG`, `TER_L_FRAG` or `TER_T_FRAG`, taking a very long time in the GPU process. Three.js r128 waits for each compile synchronously.
   - *Tests:* the `?diag` readout names the step. Changing ANGLE's backend in `chrome://flags/#use-angle` (OpenGL, D3D11on12, Vulkan) and seeing whether the freeze moves or vanishes would separate a Direct3D-specific problem from a general one.
4. **A GPU timeout or reset.** Check the "Log Messages" at the bottom of `chrome://gpu`, and Windows Event Viewer for display driver events (Event ID 4101). The page has no `webglcontextlost` handler, so a lost context would go unnoticed. Logging it in the diagnostics would settle this.

## Diagnostics already in the page

Add `?diag` to the address (`https://weeblabs.github.io/Nightscape/?diag`, or the local equivalent). A text box in the top-left shows:

- **The graphics setup:** the GPU and ANGLE backend Chrome reports.
- **During loading:** the step about to run. If the page freezes, the box shows the stuck step.
- **After loading:** the slow steps, with each step's call time and the gap before its frame. A long call means JavaScript was blocked in that step; a long gap means frames stopped arriving. The full table is also logged to the console with `console.table`.

Switching tabs adds a long gap of its own at the step where the page stuck, so note the time spent away.

Also gather:

- the "Graphics Feature Status" and "Driver Information" sections of `chrome://gpu`;
- a DevTools Performance recording of a load;
- whether the freeze also happens in Edge, which uses the same engine, and in Firefox.

## Requirements for any fix

- **Nothing may change visually.** Prove it by rendering the previous commit and the fixed version side by side, at the same frozen moments, and comparing every pixel. The Mac session did it like this:
  - **Two test copies of `index.html`:** one from the previous commit (`git show HEAD:index.html`) and one from the working tree.
  - **Frozen debug settings:** each copy sets `window.__cloudDebug` from the URL hash before the main script runs, for example `{"t":-60,"frozen":true,"frames":8,"scale":0.5}`.
  - **Seeded randomness:** each copy replaces `Math.random` with the same seeded generator in a script at the top of `<head>`, so lightning strikes match.
  - **Manual stepping:** each copy exposes `window.__tick = t => loop(t)` after the line `requestAnimationFrame(loop);`, so the loop can be stepped without the browser drawing frames. A hidden or background page does not run `requestAnimationFrame`.
  - **Comparison page:** a page holds both copies in iframes. It steps each until `window.__done`, then calls `window.__cloudScene.step()` and copies each canvas into a 2D canvas to compare pixels. `__cloudScene.setTime(…)` moves the flight on, and `__cloudScene.strike('cg', 0.35)` freezes a lightning strike at an age.
  - **Cases covered:** the default view, a later point, a ground strike, a cloud flash, and the slow-machine path with the smallest bake strips. Every comparison came out at 0 differing pixels.
- **Keep the startup behaviour:** the loading bar, then the title over the blurred scene. `?notitle` and `?diag` must keep working.
- **Delete test copies** before committing.

## Repository conventions

- Commit on `main`. Ask the user before creating any branch.
- Commit as the repository's configured author, Weeb Labs. Do not add `Co-Authored-By` lines or any mention of Claude to commits; this is a public repository.
- For visual results, show the user screenshots and ask them to look, rather than judging appearance yourself.
- Update `README.md` when behaviour or options change, and keep comments in `index.html` explaining why the code does what it does.
- Push only when the user asks. Pushing `main` updates the live demo.

## Findings

*(Add what you find here: the `?diag` output, `chrome://gpu` details, which test changed what, and the fix.)*
