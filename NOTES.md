# Asteroids — project notes (for context recovery)

Single-file HTML5 game in `index.html` (~66KB, ~1276 lines). Arcade cabinet bezel wraps
the game screen; the bezel art is a photo of the original Asteroids cabinet decal kit,
restored/enhanced (Krea used once for the big art restore).

## Key files
- `index.html` — the whole game + shaders. One inline `<script>` at end of body.
- `bezel.webp` (~2.1MB) — bezel art + **intentional veil alpha**: the screen-hole region
  is 80% opaque (alpha 1–127, mean ~26) so the baked-in reflections in the art show
  through over the game. **Never zero that alpha.** Falls back to `bezel.png` on error.
- `reflect.webp` — additive pulsing reflection overlay (`#reflect` img,
  `mix-blend-mode: screen`, `@keyframes reflectPulse`). Cropped to hole width (690 of
  1024) and made square; positioned via the % values in the `#reflect` CSS block
  (x 185–874, y 178–868 of 1024). Black background, high contrast — screen blend =
  additive. This is what Rex calls the "baked-in reflection".
- `hole.png` — Rex's clean binary hole shape (208KB).
- `ref/glow_mask.png` — **packed RGBA mask, 512×512** (R = plastic, 0=hole/255=plastic,
  G = distance field, B=0). The game loads ONLY the embedded `GLOW_FALLBACK_MASK`
  base64 — this file is a disk reference copy. 2026-11-05: replaced with the CLEAN
  packed mask decoded verbatim from `GLOW_FALLBACK_MASK` (the older RGB version had
  test-scratch residue in its B channel — 60,608 px — a false lead when Rex flagged
  the streaks). Verified 2026-11-03 (RGB era): R channel pixel-identical to the
  baked mask, distance field exact.
- `ref/glow_mask_vis.png` — GONE (Rex deleted it; replaced by `ref/glow_mask.png`).
- `ref/hole_v2.png` — earlier auto-traced hole shape (superseded by Rex's vis fix).
- `reflect.webp`, `bezel.png` also at repo root (bezel.png is the large fallback).

## The glow ("edge reflections") — how it works
Mega-Bezel-style screen bleed: game screen is blurred and smeared onto the plastic
around the hole when bright objects fly off screen. Pipeline in `index.html`:
- `GLOW = { intensity: 0.8, maxDist: 96, on: true }` (intensity bumped 0.65 → 0.8 on
  2026-11-05 — thin asteroids' glow was hard to see). maxDist = panel px.
- Screen rect in 1024 panel px: left 186, top 201, 688×647 (in COMP_FS:
  `guv = (vUv*1024.0 - vec2(186.0,201.0)) / vec2(688.0,647.0)`).
- Mask = 512×512 texture, 1 texel = 2 panel px. Packed: **R = plastic mask,
  G = clamp(dist/48, 0..1)** (48 = maxDist/2), B=0, A=255.
- `GLOW_FALLBACK_MASK` (~line 1136) = the pre-baked mask as a 19,500-char raw base64
  string. Baked from Rex's fixed `ref/glow_mask_vis.png` (R channel) with a
  recomputed chamfer distance field. **This is the single source of truth for every
  browser.** Re-bake it if the bezel art / mask shape changes.
- `bakeGlowMask()` **always** calls `bakeGlowMaskFallback()` (live alpha bake was
  deliberately killed — it disagreed between browsers). Live-bake code
  (`uploadGlowMask`) still exists but is dead.
- **CRITICAL:** the base64 is RAW — `bakeGlowMaskFallback` must do
  `fb.src = "data:image/png;base64," + GLOW_FALLBACK_MASK;`. The prefix was missing
  once, the browser 404'd a file named `iVBORw0KGgo...`, and the glow silently died
  (`maskBaked` never went true). This was the 2026-11-03 "edge reflections off" bug.
- `glowRender()` no-ops unless `glowGL && GLOW.on && maskBaked && !isContextLost()`.
- Composite shader `COMP_FS` (~line 1027): `rim = exp(-dist*14)` hot rim at the glass
  (multiplier `0.8 + 0.6*rim`), radial fade `pow(1-dist,1.7)`, CLAMP_TO_EDGE smearing
  for off-screen bleed. **Alpha is capped at 0.7** (`a *= 0.7`) — glow is light, not paint.
- **`#glow` uses CSS `mix-blend-mode: screen`** (2026-11-03 fix): the glow canvas
  composites over the bezel art; with plain src-over the smear was PAINTED onto the
  asteroid decal art (mask marks the blobs as plastic) — bright game edge = bright
  line/highlight over the art, dark edge = art dimmed ~35%. Screen blend means the
  glow can only brighten, never dim or overpaint. See gotchas below.

### Re-baking the mask (pipeline, exact)
From a 512×512 RGB vis where R = plastic mask (Rex's `ref/glow_mask_vis.png`):
1. Distance seed: `R<=127 → 0 else 1e9` (hole pixels seed it).
2. Two-pass chamfer, 8-neighbor, D8 = 1.41421356 (same JS algorithm as
   `uploadGlowMask`).
3. Pack: `R=mask, G = d>=48 ? 255 : min(255, round(d/48*255)), B=0, A=255`.
4. Save 512×512 RGBA PNG → base64 → embed in `GLOW_FALLBACK_MASK`.
   **Never paste the 19,500-char string into edit_file_tool** — do the swap in
   Python (find `const GLOW_FALLBACK_MASK = "`, replace up to `";`), then copy back.
   After copying, re-read index.html before any edit_file_tool.
5. Vis encoding (to regenerate a human-viewable vis from a packed mask):
   `R=mask, G=int(mask*(0.15+0.85*fall)), B=int(40*fall*mask/255)` where
   `fall = 1 - min(d,48)/48`; hole = black. Save under a NEW name, never over
   `ref/glow_mask_vis.png`.

## CRT shader (the `#fx` canvas, mode 2 = "Glass TV")
- `u_refl` — procedural glass reflections (sheen + moving band + corner light).
  **Currently OFF** at Rex's request — he preferred the baked-in reflection
  (`reflect.webp`); the procedural one added a uniform highlight. Toggle exists
  (`setCRT` / C key cycles pass-through → CRT → Glass).
- `u_comb` — RGB channel separation. **Reduced to 50%** (was overwhelming).
- Barrel distortion and glow kept at full.
- Scanlines kept.
- Note: `#reflect` overlay is a plain DOM img with CSS screen blend, so it is NOT
  affected by the CRT shader (Rex asked; answer: no).

## Plexiglass wear layer (2026-11-04)
Transparent "worn glass" layer over the screen: curved stress cracks in the
material, straight hairline scratches on the surface, dust specks. Lit from
below — only bright screen pixels reveal the wear (black screen = clean glass).
- **2026-11-05 — source switched to the Krea texture (current truth).** The
  procedural v5/v6 embed's bright "hero" scratches drew harsh straight lines
  on bright screens (Rex's streak report). Traced by layer isolation: with the
  DOM plexi/reflect/bezel blacked out and the bezel alpha hole cleared, the
  lines survived and appeared ONLY in Glass mode — the CRT tone curve boosts
  midtones (pluma ~3×) before the wear pass, so the EMBEDDED texture (not any
  disk file) was the culprit. `PLEXI_TEX` is now the approved Krea
  `plexi.webp` (1024×1024 black-bg, Rex-edited in Photoshop; real copy kept at
  `ref/plexi.webp`): screen-box crop (179,194,881,855) → 1024×963 LANCZOS,
  LUT `L' = 128 + v*0.75*127/255` (v = 0..255 brightness; deliberately softer
  than v6's ×2 stretch), WEBP q85 → ~51KB / 68k-char base64. Strength knob:
  `PLEXI_AMT` (2.2; lower toward 1.2–1.5 if still strong, 0 = off). The v5/v6
  procedural pipeline below is history — superseded.
- **Embedded as a base64 data URL** (`PLEXI_TEX` const in index.html). This
  is mandatory, not style: for a file:// page, Chromium treats a file://
  image as cross-origin (opaque origin) and `texImage2D` on the tainted
  image THROWS SecurityError ("The image element contains cross-origin
data"), so in any plain browser / in-app browser `plexiReady` never turned
  true and the effect was silently absent — the 2026-11-04 "I'm not seeing
  it" report. A data URL is same-origin everywhere. Same pattern as
  `GLOW_FALLBACK_MASK`.
- Disk source of truth (NOT loaded by the game): `plexi.webp` (~11KB) /
  `plexi.png`. v5 (2026-11-04, thick lines so wear survives narrow
  in-app-browser widths): 1024×963 grayscale (same aspect as the screen
  window), 50% gray = clean, brighter = scratch/dust, a few faint dark
  blobs. Python/PIL, seed 20261104, 2× supersampled 2048×1926 → LANCZOS:
  10 long off-center arc cracks (w4–6), 5 short arcs (w3), 55 straight
  hairlines (w3–4, biased shallow), one dense worn cluster at ~28%/68% (w3),
  6 full-length "hero" scratches (w5–6, brighter 158–185), 300 dust specks
  (r1–4), 14 faint dark blobs, ±6 micro-noise haze, GaussianBlur 0.7.
  Reference photo in the chat that kicked this off.
- **Contrast LUT (v6, 2026-11-04):** the EMBEDDED texture is the disk v5
  with `L' = clamp(128 + (L-128)*2.0)` applied — wear stretched toward white
  (disk v5 wear 158–185 → ~240+). Why: at Rex's pane scale (~450 CSS px
  screen) the ±0.2 wear deviation was below his perception threshold; a
  black/white 8×8 checkerboard test (±0.5) was instantly visible, proving
  the pipeline alive and the texture too subtle. The LUT lives ONLY in the
  embed step (disk files untouched — they are untracked, no git safety
  net). Re-embed = read plexi.png → apply LUT → save WEBP q85 (≈19KB,
  don't PNG it: 314KB/500k base64 chars) → swap the line in Python.
- Re-embed recipe (after regenerating `plexi.webp`): base64-encode it,
  replace the string between `const PLEXI_TEX = "` and `";` IN PYTHON (≈14k
  chars — never paste into edit_file_tool), then `node --check` the
  extracted inline script. (A white 1×1 data URL makes a good pipe test:
  screen brightens on all bright pixels; 50% gray 1×1 = zero-effect
  control. Freeze frames for A/B by patching requestAnimationFrame to
  stop after N callbacks in a temp copy.)
- Rendered in the CRT shader (`#fx` canvas), NOT a DOM layer: first attempt
  was an `<img>` with `mix-blend-mode: overlay`, but overlay saturates on
  bright pixels (1-2(1-a)(1-b) → ~no change) and this game is high-contrast
  vector (pure black + thin bright lines, almost no mid-tones), so the wear
  was nearly invisible. Shader term (end of FS, after tone-map, in ALL
  modes), current recipe (2026-11-04): local scene light only —
  `float pluma = max(max(col.r,col.g),col.b);` then
  `col += (texture2D(u_plexi, vUv).g - 0.5) * u_plexiAmt * pluma;` — wear.
  HISTORY: the light-BLEED experiments are all dead, per Rex ("wrong
  direction"): (a) glow8 8-tap rings → "thin copies of the objects sliced
  across the scratch"; (b) `halo5()` two-scale 5×5 binomial Gaussian
  (2px/10px tap spacing, fully unrolled, barrel-corrected coords) + drive
  floor → "not very good, not enough glow from thin lines"; (c) `max5()`
  morphological-dilation core + Gaussian falloff → reverted same day,
  never approved. (d) edge-GLOW scratch (same day): a constant warm rim band in the fx pass (`exp(-dE*9)` from the nearest edge, glass mode only) + the bezel halo modulated by the plexi in the glow pass via mirrored UVs (`1-||guv|-1|`) and a second plexi upload into the glow context → Rex: "no I don't like that either" — fully reverted (band, glow-pass term, and dual-context plumbing all removed; plexi uploads to the fx context only). Lessons: the raw game canvas is pure black at the screen edges, so a content-sampled edge glow is invisible; a constant rim band read as a tinted border, not glass. The bleed functions are deleted from the FS; if a new
  approach is tried, start from this clean local-light term. Why bleed was
  hard: the game is pure black + thin 1–2px bright lines — almost no
  mid-tones, and bright pixels already clamp at ~1.0, so there is very
  little source energy for any blur/dilate to work with.
  fixed in glass coords (vUv), not barrel-distorted. `PLEXI_AMT = 2.2`
  (bumped 0.4→…→1.2→2.2; 2.2 is the "obvious on bright pixels" setting,
  still to be dialed to taste with Rex); plexi uploads to texture unit 1 via
  `loadPlexi()`, `plexiReady` gates it. Because the plexi lives in the fx
  pass, `fxRender` no longer early-returns in CRT mode 0 and `setCRT` keeps
  the fx canvas visible in every mode ("CRT: Off" = pass-through + glass
  wear).
- Context-restore handler reloads plexi (`plexiReady=false` + `loadPlexi()`),
  same as the glow mask re-bake.
- (e) Full-cabinet plexi layer (2026-11-04; Rex: "expand the scratch
  layer to fit the whole screen, the whole screen includes the static
  bezel art, dont make a new texture, make the blend mode the same as
  the glowing reflection that pulses slowly"): DOM `<img id="plexi">`
  placed AFTER `#glow` (topmost in #bezel, physically the glass in front
  of everything). CSS: absolute 100%x100% of the panel, `mix-blend-mode:
  screen` (same as #reflect), `opacity: 0.22` (the haze knob),
  `pointer-events:none`. JS: one line, `document.getElementById(
  "plexi").src = PLEXI_TEX;` right before the plexiReady/plexiImg decls -
  reuses the embedded data URL, no new texture. Texture is 1024x963
  stretched to the 1024x1024 panel (~7% vertical stretch, fine for the
  experiment). The in-shader local-light wear term in the hole is KEPT
  (approved effect), so the screen hole now has BOTH: static screen-
  blend wear (DOM) + dynamic lit-from-below wear (shader). The two
  scratch mappings do NOT align (DOM spans the whole panel, shader maps
  vUv over the hole); if it reads as double, drop one (delete the img or
  the shader term). Both render paths verified same day:
  fxtest/pfull_{gpu,cpu}.html, shots shot_pfull_{gpu,cpu}.png, clean.
  Rex: expanded layer "look ok", texture needs to be more realistic.
  Follow-ups same day: (1) DOM layer switched to load plexi.webp from
  disk (marked-TEMP JS line, img onerror falls back to plexi.png) so
  texture edits show on reload without re-embedding; the WebGL shader
  still uses the embedded PLEXI_TEX (file:// images cannot be uploaded
  to WebGL - SecurityError). (2) reflectPulse animation moved off
  #reflect (now static, full opacity) onto #plexi as plexiPulse: 2.6s
  ease-in-out infinite, 0.10 to 0.22 - the old 45%-to-100% relative
  depth scaled to the plexi haze level. When the texture is final: revert
  the TEMP line to PLEXI_TEX and run the re-embed recipe.
- Debug recipe that caught the SecurityError: append
  `<script>window.addEventListener("error",e=>{document.title="ERR: "+e.message+" @"+e.lineno;});</script>`
  to a temp copy, run `msedge --headless=new ... --dump-dom`, grep
  `<title>`.
- **VERIFY BOTH RENDER PATHS (2026-11-04, cost a full round of "it's
  broken" reports):** the screenshot recipe runs with
  `--disable-gpu --enable-unsafe-swiftshader` (CPU GL) which ACCEPTS
  GLSL that the real GPU path REJECTS. Symptoms when only CPU-verified:
  FX canvas dead in Rex's browser, everything shader-based gone, only DOM
  overlays (reflect pulse) alive, crtlabel reads "No WebGL". Real-driver
  rejects hit so far: `int` function params/loop vars in the FS (swiftshader
  silently coerced to float — then `i == 0` fails as float==int) and an
  implicit vec2→float conversion at a call site (swiftshader accepted,
  glslang "no matching overloaded function"). Rules: fragment shaders get
  no `int` at all (uniforms OK — `u_mode` is int and fine), no float
  for-loops (out of spec) → unroll taps; pass explicitly typed floats.
  Verify recipe: patch a temp copy so makeShader failures set
  `document.title = "FXFAIL: "+log`, run `msedge --headless=new` WITHOUT
  `--disable-gpu --dump-dom`, grep the `<title>` AND
  `id="crtlabel"[^>]*>[^<]*` (must say Glass/Arcade/Off, never
  "No WebGL").

## History / gotchas
- ~~Browser cache shows a STALE bezel/mask~~ — **DISPROVEN** (2026-11-03). The real
  "straight left line / bright highlight on the left" bug: the `#glow` canvas
  src-over-composits its smear ON TOP of the bezel art. The mask correctly marks the
  asteroid-blob decal (which sits 5–90px outside the hole, on the plastic side) as
  plastic, so the blurred game-edge smear was painted over the art with alpha 0.5–0.9:
  bright game edge → bright line/highlight (hugging the squircle's STRAIGHT left
  section, wavy at the lobes — that's where the "straight line" look came from);
  dark edge → semi-transparent dark paint dimmed the art ~35% (measured in Rex's
  ON/OFF shots). Fix = `mix-blend-mode: screen` on `#glow` + alpha cap 0.7 + softer
  rim. Mask itself was always correct; do NOT re-bake it for this symptom.
  (File:// caching is still a mild hazard in general, but it was not this bug.)
- Effect-parity requirement: anything visual must work identically in the Bionic
  sidebar in-app browser AND a regular browser (file://). Past bug: live alpha bake
  / canvas taint differences. The pre-baked-mask design exists to kill this class of
  bug.
- All testing toggle buttons were removed at Rex's request — keep the code clean,
  no debug UI.
- Hi-Score text moved from bottom to top on the "Press any key" screen (Rex request).
- 1.2s input lockout on game-over (`gameoverLock`) — dying on the last life
  while mashing fire used to instantly restart the game. Keys/taps ignored for
  1.2s after final death; held-key repeats never restart (Rex request).
- Game screen was made more square (non-trational for Asteroids) to fit the bezel
  hole better (Rex doesn't care about tradition, lol).
- Krea AI MCP: configured via env var API key. Rex's subscription credits are scarce —
  use Krea only for special jobs; local image gen (Wan2GP) first. There is a
  `krea-restoration` skill for restoring photo-based game assets.
- ShaderGlass is installed at `C:\Users\rex\tools\ShaderGlass` (reference only; we
  hand-rolled the effects).

## Environment quirks
- run_python sandbox: no numpy; Pillow yes. Stage host files via stage_files with
  HOST paths. Outputs land in scratchpad; use `copy` tool to move into working dir
  (can_overwrite=true to replace index.html).
- `node --check` on the extracted inline script validates syntax
  (regex `<script>(.*)</script>` re.S, first match = the game script).
- Headless testing (great for verifying this page):
  `"/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe" --headless=new
  --disable-gpu --no-first-run --no-default-browser-check --user-data-dir=<dir>
  --enable-unsafe-swiftshader --virtual-time-budget=5000 --dump-dom file:///...`
  (needs --user-data-dir; combine with --screenshot for visuals; virtual time makes
  timers/image loads resolve).
- App browser refresh: `open_url_in_app_browser` with
  `file:///E:/Bionic%20Projects/Games/Asteroids/index.html`.
- Git Bash warns "could not find /tmp" — harmless; don't write to /tmp.
- curl needs `--ssl-no-revoke`.
- Project has .git; last commit `42158ff` (pre-glow). Current index.html/bezel
  uncommitted.

## Verification recipe (proven)
- `run_python`: decode `GLOW_FALLBACK_MASK` from index.html (regex
  `const GLOW_FALLBACK_MASK = "(.*)";` re.S), compare R with `glow_mask.png` and
  recompute the chamfer field to compare G. 2026-11-03: 0 differing pixels both ways.
- Headless render: `msedge --headless=new ... --screenshot` (see env quirks) — the
  2026-11-03 render showed the correct wavy left edge + glow following the lobes.

## Rex's preferences
- Casual tone, quick, direct. He laughs at iterative debugging ("lol", "dude", "!!").
- He re-cuts bezel alpha himself when he wants.
- One mask / one source of truth; identical behavior everywhere.
