# Freerun 08

A polished third-person parkour time trial across a long training course suspended over reflective
blue water: sprint, slide, jump, vault, roll, wall-run, and wall-jump through eight ordered
checkpoint gates. `prompt.md` is a measured spec of the finished game — given to a coding agent as
a single message, it one-shots the baseline unattended. The Acceptance checklist at the bottom of
the prompt defines done.

▶ **[Play the baseline](https://freerun-08.netlify.app/)**

| Model | Harness | Time | Outcome |
|---|---|---|---|
| GPT-5.6-Sol | Codex CLI | ~36 min | Selected baseline |
| Claude Fable 5 | Claude Code | ~159 min | Completed, not selected |
| Grok 4.5 | Cursor CLI | ~21 min | Completed, rejected |

| Tooling | Date |
|---|---|
| npm, Vite, TypeScript, PlayCanvas 2.20+ | 2026-07-16 |

## Starting point the prompt assumes

A barebones Vite + TypeScript app with `playcanvas` installed and wired: a full-window
`<canvas id="application-canvas">`, module entry `/src/app.ts`, dev server on `127.0.0.1`.
The prompt targets PlayCanvas directly and requires engine 2.20 or newer.

## Assets

The prompt builds exclusively from `public/assets/quaternius-urban-v1/`, a CC0 pack by
[Quaternius](https://quaternius.com/) converted to GLB:

| Sub-pack | Source | Notes |
|---|---|---|
| `downtown-city` | [Downtown City MegaKit](https://quaternius.itch.io/downtown-city-megakit) | Concrete, metal-concrete, and trim PBR textures used on the authored course primitives |
| `base-characters` | [Universal Base Characters](https://quaternius.itch.io/universal-base-characters) | `Male_Ranger.glb` hooded full-body hero |
| `animation-library` | [Universal Animation Library](https://quaternius.itch.io/universal-animation-library) | UAL1 + UAL2 locomotion and parkour clips bound to the hero by bone name |
