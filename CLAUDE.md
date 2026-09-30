# MMM-Chladni — context for Claude sessions

Charles's MagicMirror² module: Chladni figures, sand on a vibrating square plate, for his
hallway mirror, as one page in a rotation of pages. Split out of MMM-ChaosTheory on 2026-09-28
with its history (it was the `chladni` simulation of that module's fractal page).

## Files

- `MMM-Chladni.js` — module shell: one canvas plus an HTML caption (equations + live readout,
  updated 2×/s). New sim every `cycleSeconds` and on each `resume()`. Loop: `setTimeout` until a
  frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a sim with
  `resting = true` is polled only every 500 ms; while MagicMirror fades the module out (`hidden`
  is set at the start, `suspend()` comes after), frames draw nothing. `turns: { of, at }`: only
  every nth showing; otherwise the wrapper gets `display: none` and nothing starts.
- `simulations/chladni.js` — the `chladni` sim on `window.ChladniSimulations`. Grains hop where
  the amplitude exceeds a threshold (farther the more it moves, biased downhill), creep onto the
  nodal line below it, fall off the edge. 49 figures (a deck, shared by successive instances),
  five stories (a deck). Pixel buffer with dirty-rect `putImageData`; rests once settled.
- `simulations/plate.js` (`window.ChladniPlate`) — free–free beam modes and Ritz's method for the
  free square plate. `tools/chladni-modes.js` precomputes the modes (λ ≤ 470, ν = 0.3, 12 beam
  modes per parity each way) into `data/chladni-modes.js` (`window.ChladniModes`), so the Pi
  never solves anything. Re-run it after changing `plate.js`; a test checks the file against a
  fresh solve.
- UMD-style, so the physics runs in Node: `tests/chladni.test.js` (`node --test`, no
  dependencies; frequencies match Leissa's published values).
- `node_helper.js` — the stats panel (`statsPanel: true`): CPU of Electron and cage, per core,
  temperature, from `/proc`, only while shown.
- `dev/preview.html` — runs the module in a desktop browser (`python3 -m http.server` in the repo,
  then `/dev/preview.html?figure=12&seed=1`).

The shell (`MMM-Chladni.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is shared in
spirit with the sibling modules (MMM-ChaosTheory, MMM-FractalZoom, MMM-Atom, MMM-SacredGeometry,
MMM-Tilings, MMM-PlanetsDance, MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck, MMM-StandardMap, MMM-ChaoticWaterwheel, MMM-DoubleSlit, MMM-Sandpile, MMM-Harmonograph): a fix there
probably belongs in the siblings too.

On the mirror it shares the fractal page with MMM-FractalZoom, taking turns
(`turns: { of: 2, at: 1 }` here, `{ of: 2, at: 0 }` there).

## Measured cost on the Pi

700², 12 fps, seven showings, Electron + cage, stats panel on: ~108% of a core while the sand
moves (~12 s, at 11 fps; ~75% of the canvas changes per frame), ~5% settled, 53% (47–61) over a
30 s showing, its page change included (~154% for the first 3 s). Hidden: 0.3% (baseline 0.2%).

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame).
- So: keep each frame's changes spatially compact, and rest when the picture is static. Line
  width, opacity, `rAF` vs timer made no difference.
- MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be set
  that way; use `debugStats: true` and a `grim` screenshot to see fps on the Pi.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. **Measure, don't guess**: on the Pi,
  `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS over 60 s. Record before/after numbers
  in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). `suspend()` and
  `resume()` must fire on page changes, or the loop burns CPU 24/7.

## Deploying and testing

- This repo is public so the Pi can `git clone`/`git pull` without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-Chladni`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: run it in a desktop browser (`dev/preview.html`), then confirm performance
  on the Pi.
- Commit as Charles's GitHub noreply address (set in this repo's git config).
