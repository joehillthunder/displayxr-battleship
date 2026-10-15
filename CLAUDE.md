# CLAUDE.md

Battleship 3D: the classic two-player game on a golden-hour sea. It runs as glasses-free 3D on Leia
displays through the DisplayXR inline-3D SDK, and in 2D in any browser. See `README.md` for the
player-facing description.

- Live: https://joehillthunder.github.io/displayxr-battleship/
- Deploy: push to `main`. GitHub Pages serves the repo root as-is (legacy Pages + `.nojekyll`; no
  workflow, because the gh token has no `workflow` scope). It's live about 30 s later.
- One of the DisplayXR HQ demos. Its room is in the Arcade wing of the hub repo
  `joehillthunder/joehillthunder.github.io` (`home/demos.js` id `battleship`, set piece `battleship(g)`
  in `home/office.js`). The old hub path `displayxr-demos/apps/battleship/` is a redirect stub that
  keeps `?query`. If the URL or the description changes, update the hub entry and `demo.json` too.

## Stack and constraints

- Plain ES modules, no bundler, no package.json. three.js `0.161.0` and `@displayxr/inline3d` `1.45.1`
  load from jsDelivr through the import map in `index.html`. Pin exact, published versions. PeerJS
  `1.5.4` is loaded lazily from jsDelivr, only for online play (`js/net.js`).
- No asset files. Ships, textures (canvas-drawn), particles and sound (WebAudio) are all generated in code.
- Units: one grid square = 1 m, y is up, the waterline is y = 0, and the camera looks toward -z.
  Row A is at -z (far), column 1 at -x. Ship models are built along +x with the bow at +x. A
  vertical ship (`dir: 'v'`) is rotated `-π/2` about y so it runs along +z.
- Players: `team` 0 is blue, 1 is red (`ships.js` `TEAM`). Online, the host is blue and fires first.
- No Node and no Python on this PC. Use PowerShell or perl for scripting.

## Architecture (`js/`)

| File | Role |
|---|---|
| `main.js` | Game flow and UI: menu → placement → turns → game over, for the `local`, `ai`, `online` and `watch` modes. Input, HUD, dialogs. |
| `world.js` | The scene: camera views (`camTo('home' \| 'target' \| 'both' \| 'place' \| 'orbit')`), picking, `setViewer`, strike sequences (`outgoing`, `incoming`, `impact`, `sink`, `victory`). |
| `stage.js` | Renderer and DisplayXR session. Camera rig with the attach pattern: eye cameras are children of `stage.cam`, and `setViewRig` runs every frame. 2D fallback loop. |
| `board.js` | `Board`: holographic grid shader (per-square tint `DataTexture`, sonar sweep), labels, buoys, ships riding the swell, hit/miss markers, wrecks. `Reticle`. |
| `ships.js` | Procedural ships: lofted hull (vertex colours), turrets (`userData.turrets/guns/muzzle`), masts, funnels, carrier deck, submarine. `charModel` for wrecks, `ghostModel` for placement. |
| `fx.js` | Pooled point sprites (`smoke`, `spray`, `add` pools), burning emitters, missiles, explosions, splashes, shockwaves, debris, fireworks, floating banners, camera shake. |
| `ocean.js` | Sky dome and ocean shaders. The waves are a sum of sines with the same `WAVES` table in GLSL and on the CPU (`waveHeight` / `waveSlope`). |
| `game.js` | Rules (`Fleet`, `Player`, `applyShot`) and the AI (`aiPick`: probability density over the remaining ship placements). No three.js. |
| `net.js` | PeerJS wrapper. The room code is the host's peer id `dxr-battleship-<CODE>`. |
| `audio.js`, `util.js` | Synth SFX, maths helpers |

### Key patterns

- **Viewer vs turn.** `G.viewer` is whose fleet is on the left (`home`) board. `G.turn` is who fires
  next. The shooter's shots animate as `outgoing` (home → target), and shots at the viewer as
  `incoming` onto home. `world.setViewer(me, foe)` rebuilds both boards from what `me` knows.
- **Pass & Play secrecy.** `handoff()` wipes both boards before the dialog appears. A player's
  `received` shots past `seen` are replayed (`replayUnseen`) when they take the screen.
- **One queue.** Shots and turn changes run through `enqueue()` in `main.js`, one at a time. Online,
  the opponent can fire while your own impact is still animating. Don't call `turnStart` from a bare
  `setTimeout`. `G.gen` bumps on every new game, so a stale async chain must check it and stop.
- **Online protocol** (`onNet`): `hello {name, rules}` → both place → `placed` → `shot {r,c}` /
  `result {r,c,hit,sunk,allSunk}` → at the end, `fleet {ships}` (so the loser sees the remaining
  ships) and `rematch`. Each side keeps its fleet secret and answers shots itself.
- **Ships stand up out of the glass.** `SHIP_LIFT` in `ships.js` (1.6) stretches every model vertically
  (and 1.22 across) so the ships read as solid and stereo-pop; turrets, bridges and masts are taller than
  life on purpose. Each ship has a contact shadow + foam collar (`makeShadow` in `board.js`) and casts
  real shadows on itself (one 2048 sun shadow map per frame, shared by both eyes). Placed ships hover at
  `DROP_H` (out of the glass) as a ghost, then drop in with a splash (`addShip(..., { drop: true })`).
- **Effects are pooled.** Spawn through `fx.add/smoke/spray.spawn(...)`, and run per-frame work with
  `fx.task(fn)` (return `false` when done). Point-sprite size is in world units: the shader scales
  it by the per-eye viewport height, set through `stage.onEye`.

## DisplayXR / stereo rules

- One persistent canvas with `will-change: transform` from first paint, never remounted. The woven
  buffer is double width, sized by hand with `setPixelRatio(1)`, and each eye is capped at 2560 px.
- The `#boot` cover stays on until `firstWoven`, then comes off with a hard cut, never a fade.
- No `filter`, `opacity`, `backdrop-filter`, border or shadow on the canvas or its ancestors, and no
  `backdrop-filter` on anything over it. Dialogs are centred cards, never full-screen backdrops.
- Depth: convergence = the camera's distance to the board in view (`POP` 1.0, so the board sits on
  the glass). `ipdFactorFor` scales the eye separation with distance (capped at 5), and the Depth
  slider sets a gain from 0 to 2. Anything that rises (missile arcs, debris, banners) comes out of the glass.
- In `xrFrame`, validate the views and viewports *before* clearing.

## Run and check locally

```sh
# any static server on the repo root, for example a PowerShell HttpListener, then:
chrome --headless=new --use-angle=d3d11 --user-data-dir=<tmp> --remote-debugging-port=9344 \
  --window-size=1400,788 --enable-logging=stderr --v=0 "http://localhost:<port>/?watch"
```

- `window.battleship = { G, world }`. Drive the game over CDP (`Runtime.evaluate`): click the menu
  buttons (`[data-mode=ai]`, `#random`, `#ready`, `#handoff-go`), aim with `G.cursor = {r, c}`, and
  fire with `dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter' }))`. Grab frames with
  `Page.captureScreenshot` or `canvas.toDataURL()` (`preserveDrawingBuffer` is on).
- `--virtual-time-budget` doesn't advance requestAnimationFrame, so the battle never progresses
  that way. Use real-time headless plus CDP.
- A background tab pauses rAF. Test online play with **two separate browser instances**, not two tabs.
- On this PC, Chrome can't reach `0.peerjs.com` (fetch fails), while Edge and curl can. Test online
  play in Edge.
- Don't evaluate `location.reload()` over CDP: the socket hangs. Start a fresh browser instead.
- URL params: `?watch` or `?auto` (computer vs computer, loops), `?join=CODE` (opens the Online dialog).
- Headless only exercises the 2D path. Stereo needs the Leia panel in the DisplayXR Browser
  (`C:\Program Files\DisplayXR\Browser\chrome.exe`, its window visible). The page logs
  `[battleship] firstWoven ...` and frame stats to the console.

Before you push, check at least: no console errors, a game vs the computer to victory, and (if you
touched turn flow or `net.js`) a pass-and-play round and an online exchange.

## Gotchas

- Every `Sfx` method must return early when `this.ctx` is null (no click yet, e.g. `?watch`): one
  unguarded call threw inside a strike and stalled the whole turn queue.
- Additive particles saturate to white fast. Keep fire alphas around 0.4–0.55 and grow the size instead.
- The ocean mesh spacing must stay well under the shortest vertex wavelength, or the far water
  aliases into stripes. Short ripples go only in the fragment normal.
- Haze starts at 35 m (`smoothstep`). Exponential fog from 0 turned the whole sea orange.
- `ShaderMaterial` shaders end with `#include <tonemapping_fragment>` and `#include <colorspace_fragment>`
  on their own lines.
- Commit messages end with a `Co-Authored-By` line when an agent writes the commit.
