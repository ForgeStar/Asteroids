# ASTEROIDS — Classic Arcade Clone

A single-file clone of Atari's 1979 vector **ASTEROIDS** in vanilla JS + canvas, rendered inside a dark vector-style cabinet bezel with the collection's CRT glass shader (barrel distortion, scanlines, reflections).

## Play

- Open [index.html](index.html) — that's the whole game (no build step, no dependencies).
- ◄/► or A/D: rotate · ▲/W: thrust · Space: fire · N: hyperspace · Any key: start
- Touch: tap to start · tap screen to fire · drag: sideways = rotate, up = thrust
- P: pause · M: mute · C: cycle CRT glass mode (Off / Arcade / Glass)

## Mechanics (per the 1979 original)

- **Asteroids** in three sizes: large **20**, medium **50**, small **100** points. A large rock splits into two medium, a medium into two small. Rocks are random jagged polygons that wrap around the screen edges.
- **Waves** start with large rocks only — 4 on wave 1, +1 per wave up to 8, and they get a little faster from wave 5 on. Clear everything to advance.
- **Ships**: 3 to start, an extra ship every 10,000 points. A destroyed ship respawns at center with 2 s of blinking invulnerability. Remaining ships shown bottom-left.
- **Fire**: max 4 bullets on screen at once, like the original.
- **Hyperspace (N)**: teleport to a random spot with a 15% chance of materializing badly — 1.2 s cooldown.
- **UFOs** from wave 2: the large saucer (**200 pts**) wanders and fires randomly; the small saucer (**1000 pts**) wobbles faster and aims at you with slight inaccuracy. Both wail the two-tone siren.
- **Score popups** appear where things die, hi-score persists in `localStorage`.
- **Sound**: Web Audio — gas-thruster noise on the engine, square-wave fire blips, filtered-noise explosions, and a two-tone UFO siren.

## Art

- Bezel (`bezel.webp`/`bezel.png`) is a generated placeholder — dark plastic, ASTEROIDS logotype, ship/rock line glyphs, coin doors. Swap in hand-made bezel art whenever one exists; the screen window is an 800×600 transparent hole at left 112 / top 212 on the 1024×1024 panel.
- A 2× internal render keeps the white line work crisp at any size.

## GitHub Pages

This repo is deployed as-is from the root, so the game lives at:

    https://forgestar.github.io/ASTEROIDS/
