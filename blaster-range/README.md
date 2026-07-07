# Blaster Range

A fast-twitch arcade first-person blaster: dusk-lit low-poly downtown block, sprint/slide/jump
movement, full-auto sci-fi rifle, patrolling mannequin training bots, walkable roofs, procedural
audio and HUD. `prompt.md` is a measured spec of the finished game — given to a coding agent as a
single message, it one-shots the baseline unattended. The Acceptance checklist at the bottom of
the prompt defines done.

▶ **[Play the baseline](https://blaster-range.netlify.app)**

| | |
|---|---|
| Model | Claude Fable 5 (`claude-fable-5`) |
| Harness | Claude Code, single message, run unattended |
| Tooling | npm, Vite, TypeScript, PlayCanvas 2.20+ |
| Baseline run | ~41 min, 86 turns |
| Date | 2026-07-07 |

## Starting point the prompt assumes

A barebones Vite + TypeScript app with `playcanvas` installed and wired: a full-window
`<canvas id="application-canvas">`, module entry `/src/app.ts`, dev server on `127.0.0.1`.
The prompt targets PlayCanvas directly and requires engine 2.20 or newer.

## Assets

The prompt builds exclusively from `public/assets/quaternius-urban-v1/`, a CC0 pack by
[Quaternius](https://quaternius.com/) converted to GLB (file names and pivots in the prompt's
`ASSET_TUNING` table were measured against these conversions):

| Sub-pack | Source | Notes |
|---|---|---|
| `downtown-city` | [Downtown City MegaKit](https://quaternius.itch.io/downtown-city-megakit) | glTF repacked to GLB, shared texture atlases kept external |
| `animation-library` | [Universal Animation Library](https://quaternius.itch.io/universal-animation-library) | UAL1 + Mannequin_F GLBs used as shipped |
| `sci-fi-guns` | [Sci-Fi Essentials Kit](https://quaternius.itch.io/sci-fi-essentials-kit) | weapons subset, glTF repacked to GLB |
