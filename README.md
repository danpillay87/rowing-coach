# GHOST

**Row your ghost.** A single-file, **fully on-device** indoor-rowing form coach.
Point a side-on webcam at yourself on the erg and a luminous neon figure mirrors
your stroke on a calm Monument-Valley dusk — body lean, knee angle, stroke rate,
drive:recovery ratio, consistency, and the headline fault it hunts for: **opening
your back too early on the drive.** Save your best stroke and a **ghost** of it
overlays your live figure so you can row to match the ghost of your best self.

GHOST also **sharpens its own analysis over time.** Rather than fixed thresholds,
it learns the distribution of *your* strokes and tunes its own sensitivity so it
flags only your genuinely off strokes — and when it can't see you cleanly, it says
so and softens instead of guessing.

The screen is deliberately lean: the figure is the hero, with one score read and
one coaching cue. Everything else — controls, your ghost, progress, the debug
panel — lives behind a single **⋯** menu.

It runs entirely in your browser. **It never records, uploads, or saves a
single frame.** Each camera frame is read for pose landmarks and immediately
discarded. What's kept is a handful of derived **numbers** — your session
summaries, your best-stroke shape, and what the coach has learned about its own
sensitivity — stored locally on your device. Never any video.

---

## How to run

`getUserMedia` (webcam access) and the ES-module `import` only work in a
**secure context** — that means `https://` or `http://localhost`. Opening the
file directly as `file://…/index.html` will **not** work (the camera will be
blocked and the model import may fail).

So serve the folder over localhost. From inside `rowing-coach/`:

```bash
python -m http.server 8000
```

then open **http://localhost:8000** in your browser.

(Any static server works — e.g. `npx serve`, `php -S localhost:8000`. The point
is a real `http://localhost` origin, not `file://`.)

Then:

1. Press **Start** and allow camera access when prompted.
2. Sit **side-on** to the camera, ~2–3 m back, with your **whole body in
   frame** (head to feet). Good, even lighting helps the pose model a lot.
3. Row. The neon figure tracks you; a small score read and a single coaching
   cue surface only when there's something to say. Tap **⋯** for everything
   else. Press **Stop** to release the camera.

First load fetches the pose model from a CDN (a few seconds, one network call
for the *model weights only* — never your video). After that it's all local.

### Phone

It's built mobile-first and works on a phone. Prop the phone side-on, press
Start, and use **Switch camera** to pick the rear camera if that frames you
better. On iOS you must be on `https://` or `localhost` (Safari blocks the
camera otherwise) — the easiest path is to run the server on a laptop and open
the laptop's LAN address over https, or use a localhost tunnel.

### Controls

The main screen shows only **Start / Stop** and a single **⋯** menu. The menu
slides up a sheet with everything else:

- **Start / Stop** — turns the camera and analysis on/off (always visible).
- **⋯ menu** — opens the sheet (always visible). Inside:
  - **Recalibrate** — re-learn your normal stroke without ending the session.
  - **Side: AUTO/LEFT/RIGHT** — cycles `AUTO → LEFT → RIGHT`. AUTO picks
    whichever side of your body the camera sees more clearly. Override it if you
    turn around. (Switching resets the stroke stats — the geometry changed.)
  - **Switch camera** — front/rear camera toggle.
  - **Show / Dim video** — lift or restore the dim scrim over the camera feed.
  - **Save stroke** — pin your most recent stroke as the **ghost** to chase.
  - **Replay best** — loop your ghost as a neon cartoon to study.
  - **Ghost: ON/OFF** — overlay the ghost of your best stroke on your live figure.
  - **View: Tron/Skeleton** — neon figure (default) or the raw pose skeleton.
  - **Your progress**, the **Coach** panel, **Debug & telemetry**, and the
    clear-data controls (history / ghost / **self-tuning**) all live here too.

---

## How it works

All processing is done by **Google MediaPipe Tasks Vision — PoseLandmarker**,
loaded from a CDN and running in-browser on WebAssembly (GPU delegate when the
device supports it, automatically falling back to CPU). It returns 33 body
landmarks per frame. We use a side-on subset (shoulder, hip, knee, ankle,
wrist, foot) for the maths.

### Side selection
The app sums the landmark *visibility* scores for the left vs right
shoulder/hip/knee/ankle and uses whichever side the camera sees better. This is
why you film side-on — one half of your body faces the lens. "Flip side" lets
you override when you've turned around.

### What it measures each frame
- **Body lean** — angle of the trunk (shoulder→hip line) away from vertical,
  *signed*: **positive = forward lean** (toward the catch), **negative =
  layback** (toward the finish). The sign comes from whether your shoulder is
  on the feet-side of your hip.
- **Knee angle** — the interior hip–knee–ankle angle (~180° straight legs,
  smaller = more compressed).
- A **leg-extension proxy** `0…1` derived from the knee angle (0 = fully
  compressed at the catch, 1 = legs straight).

Lean and knee are smoothed with an exponential moving average so the readouts
and the fault logic aren't twitchy. Every displayed number is rounded.

### Stroke segmentation (how it counts and splits strokes)
On an erg, your **hips slide toward the feet on the recovery** (you compress up
to the catch) and **away from the feet on the drive**. So your hip's horizontal
position is a clean repeating wave:

```
 compression  ▁▁▂▄█▄▂▁▁▂▄█▄▂▁▁     peaks = CATCH (most compressed)
                                    troughs = FINISH (most extended)
```

The app builds a "compression" signal = how close the hip is to the feet,
smooths it, and keeps a rolling ~2.5 s window. It then finds local **maxima**
(the **catch** — hips nearest the feet) and **minima** (the **finish** — hips
furthest away) using a 3-point extremum test, with two guards so noise can't
invent strokes:

- a **minimum amplitude** — if the hip is barely moving (you're not really
  rowing), no strokes are detected at all;
- a **minimum dwell time** plus a requirement that each extreme be a real
  fraction of the window's amplitude away from the previous opposite extreme.

From there:
- **Drive** = catch → finish. **Recovery** = finish → catch.
- **Stroke rate (SPM)** = 60 / (time between consecutive catches), averaged.
- **Drive:recovery ratio** = drive time vs recovery time within each stroke.
- **Consistency (0–100%)** = combines how little your **stroke length** and
  your **catch body-angle** vary across the last ~5 strokes. Tight, repeatable
  strokes score high; ragged ones score low.

### The priority fault — *early back-opening*
This is the coach-grade bit. Correct drive sequencing is **legs → back → arms**:
your back should hold its forward catch angle until the legs have done most of
their work, *then* swing open to layback. The classic fault ("opening the back
early" / "shooting the slide") is the **trunk swinging toward layback while the
legs are still bent** — the body-swing starts before the leg-drive is finished.

The app detects this **during the drive** (between a catch and the next finish):

1. At the catch it snapshots two baselines: your **trunk lean** and your
   **leg-extension** proxy at that instant.
2. As the drive progresses it continuously computes:
   - **leg progress** — how far the legs have extended from the catch toward
     straight (`0` at the catch … `1` straight);
   - **back opening** — how many degrees the trunk has already swung toward
     layback, measured from the lean it had at the catch.
3. **Fault** = the back has opened more than ~8° **while the legs are still
   less than ~60% extended.** In plain terms: meaningful body swing happened
   while the legs were still doing the work — wrong order.

When it fires, the **trunk segment of the skeleton turns amber** on the video
and the coaching cue says *"Back opening too early — drive with the legs
first."* The fault is also tracked per stroke, so if it keeps recurring the cue
escalates to *"Sequence: legs, then back"* with a count.

### One cue at a time
Rather than spamming a list, the app shows a **single prioritised cue**,
colour-coded:

> early back-opening (red) **>** rushed recovery (amber) **>** over-compression
> (amber) **>** otherwise *"Sequence looks strong"* (green)

- **Rushed recovery** fires when drive:recovery drifts above ~1.5:1 (recovery
  too quick — you want nearer 1:2).
- **Over-compression** fires when the knee closes past ~55° at the catch
  (shins driven well past vertical).

Cues are held briefly so the panel reads steadily instead of flickering.

> **Note:** the exact angles quoted above are the *cold-start defaults*. In
> practice GHOST learns your normal stroke first, coaches relative to your own
> baseline, and then tunes its own thresholds (below) — so what counts as
> "off" is personalised, not a fixed number.

### Self-calibration — learning *your* normal
On Start, GHOST spends a couple of warm-up strokes settling, then ~12 strokes
**learning your normal** (it shows a calm "learning your stroke" state, not
faults). It records the median + a robust spread (from the median absolute
deviation) of your catch/finish lean, knee range, drive/recovery timing, ratio,
sequencing onset and stroke length. From then on every stroke is scored
**0–100 against your own baseline**, and a deviation only counts when it's
genuinely unlike *your* normal.

### Self-optimisation — the coach sharpening *itself*
This is the part that improves the **algorithm**, not you. GHOST keeps a rolling,
capped record (numbers only) of your per-stroke deviations, sequencing onset,
ghost-divergence, closeness and which faults fired — **across sessions** — and
uses it to tune its own sensitivity:

- **Auto-tuned thresholds.** Instead of fixed constants, the fault/deviation
  thresholds are read off *your* accumulated distribution so faults fire at a
  sensible **target rate** — roughly your worst ~15–20% of strokes (beyond about
  your own 83rd percentile of deviation). If it's over-firing it tightens; if
  it's gone silent it loosens. It converges on useful sensitivity on its own.
- **Anti-drift.** Each tuned value is blended toward the data, **clamped to a
  band around the original default** (its anchor), and **rate-limited** to a
  small step per update — so it can self-correct but can never run away or
  oscillate. Until enough strokes are gathered it simply uses the defaults.
- **Confidence-aware.** GHOST watches how cleanly it can see you (landmark
  visibility + how steady your stroke rhythm is). When confidence drops it
  **widens its tolerance** and **softens or suppresses coaching** — showing a
  quiet *"reading you…"* rather than emitting a guess from a poor read.
- **Adapts as you improve.** Your best-stroke bar drifts slowly upward (anchored
  and capped) as you row better, so the ghost keeps challenging you.

A small **self-tuning** pip on screen shows it's learning itself and its current
confidence. The full picture — the tuned thresholds, the accumulated
distributions, the confidence model — is laid out in the **Debug & telemetry**
panel (in the **⋯** menu) for inspection, and **Reset self-tuning** in the menu
clears it back to the defaults.

---

## Privacy

- No `MediaRecorder`, no canvas capture-to-file, no uploads, no analytics.
- Frames go: **camera → pose landmarks → discarded.** They are never stored.
- The only network requests are the one-time downloads of the MediaPipe library
  and the pose-model weights from a public CDN. **Your video is never sent
  anywhere.**
- Per-frame analysis lives in a single in-memory object and is gone on reload.
- What persists is **numbers only**, in this browser's `localStorage`:
  - your learned **baseline** (medians + spreads of your stroke metrics),
  - per-session **summaries** (date, duration, strokes, avg/best score, etc.),
  - your **best-stroke shape** (the ghost — pose-normalised joint coordinates,
    never pixels), and
  - the **self-tuning** store (rolling, capped distributions of your deviations
    + the thresholds the coach derived for itself).
  None of it is video; none of it leaves the device. The menu has explicit
  **Clear history / Clear ghost / Reset self-tuning** controls.
- The on-screen badge — **"on-device · nothing saved"** — reflects that no video
  is ever recorded, uploaded, or kept.

---

## Known limitations

This is a **proof of concept**, and a single side-on webcam can only see so
much:

- **2D, single camera.** All angles are measured in the image plane. If you're
  not cleanly side-on, or you rotate, the angles skew and the side-detection
  can flip. It estimates depth poorly by design — it doesn't use 3D.
- **Requires side-on framing with your whole body visible.** If the model can't
  see your shoulder/hip/knee/ankle confidently it shows a *"reposition"* hint
  and stops computing rather than guess. Poor light, baggy clothing, a partially
  cropped body, or the erg's monitor/flywheel occluding your legs all degrade it.
- **Cannot measure force, power, or watts.** It sees *shape and timing*, not
  load. It has no idea how hard you're pulling — only how you're sequencing and
  moving. Use it alongside your erg's monitor, not instead of it.
- **It is not a physiotherapist or a qualified coach.** It learns *your*
  consistency and sequencing and tunes its own sensitivity, but it has no model
  of correct technique beyond that — treat the cues as prompts to self-check,
  not medical or definitive advice.
- Stroke detection needs a few strokes to warm up before SPM, ratio, and
  consistency settle, and very slow or very erratic rowing can confuse the
  segmentation.

---

## Tech notes

- Vanilla HTML/CSS/JS in one file. **No framework, no bundler, no build step.**
- MediaPipe Tasks Vision is pinned to **`0.10.20`** and imported as an ES module
  straight from jsDelivr; its WASM runtime is loaded from the matching
  `0.10.20/wasm` path (kept on the same version on purpose — some later CDN
  builds ship without the wasm folder).
- Pose model: `pose_landmarker_full` (Google-hosted), with automatic fallback to
  `pose_landmarker_lite` and to the CPU delegate if GPU init fails.
