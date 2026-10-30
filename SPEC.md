# ASTEROIDS — Scope Lock

**One sentence:** Classic 1979 Atari vector Asteroids — rotate, thrust, shoot, hyperspace — single-file HTML/JS in the collection's cabinet style.

## Core loop
Steer ship → dodge asteroids → shoot rocks (they split smaller, score more) → clear wave → next wave with more rocks + UFOs → ship hit = explosion, lose a ship → 3 ships → game over with hi-score.

## MVP features
- Ship: rotate (◄/► or A/D), thrust (▲/W, noise engine sound), fire (Space, max 4 bullets), hyperspace (N, small death chance)
- Asteroids: 3 sizes (20/50/100 pts), large→2 medium→2 small, random polygons, screen wrap
- Waves: start with large rocks (4, +1 per wave, cap 8), all cleared = next wave
- UFOs: from wave 2, large (200 pts, sloppy shots) and small (1000 pts, aimed shots), cross screen horizontally
- Lives: 3 ships, extra ship every 10,000 pts, respawn with brief invulnerability
- Score popups at destruction point (like the original), hi-score in localStorage
- Attract screen (title + drifting rocks), game over screen, pause (P), mute (M), CRT cycle (C)
- Cabinet: regenerated vector-style bezel (v2, with alpha-verified WebP) + the collection's WebGL CRT glass shader
- Audio: Web Audio — thrust noise, fire blip, explosion bursts, UFO siren, extra-ship jingle
- Touch: tap = start/fire, drag = steer

## Controls
◄/► or A/D rotate · ▲/W thrust · Space fire · N hyperspace · P pause · M mute · C CRT · Any key start

## Tech
Single `index.html`, vanilla JS + 2D canvas, no build step. CRT shader + audio patterns copied from LANDER/PONG.

## OUT OF SCOPE
- 2-player mode, attract-mode AI demo play, multiple difficulty variants
- Ship bank/extra-life tuning beyond 10k, UFO bullet cap beyond basics
- Custom bezel art by hand (generated placeholder bezel only)
- Mobile touch steering polish

## Tasks
- [x] 1. SPEC + bezel placeholder
- [x] 2. Shell: page, bezel CSS, CRT shader, audio plumbing
- [x] 3. Ship: rotate/thrust/fire/wrap + engine sound
- [x] 4. Asteroids: spawn, split, collide, score popups, waves
- [x] 5. Lives, extra ship, game over, attract screen
- [x] 6. UFOs + hyperspace
- [x] 7. README, commit, preview
