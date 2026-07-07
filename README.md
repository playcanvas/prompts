# PlayCanvas Prompts

Example projects, minus the code. Each folder is one game and holds exactly two files: a
`README.md` with the details (tooling, model, where the assets come from) and a `prompt.md` that
one-shots the baseline when handed to a coding agent as a single message.

| Prompt | Game | Model | Play |
|---|---|---|---|
| [blaster-range](blaster-range/) | Fast-twitch arcade FPS | Claude Fable 5 | [▶ play](https://blaster-range.netlify.app) |

## Using a prompt

1. Set up the starting point described in the prompt folder's README (Vite + TypeScript +
   PlayCanvas) and download its listed assets into `public/assets/`.
2. Paste `prompt.md` to your coding agent as the only message and let it run unattended.
3. Done = the Acceptance checklist at the bottom of the prompt passes in the running game.
