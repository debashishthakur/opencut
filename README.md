# OpenCut — personal working copy

My working copy of [OpenCut](https://github.com/OpenCut-app/OpenCut), the free and open-source video editor, plus my own experiments on top of it. Everything here is MIT-licensed; credit for the editor goes to the upstream OpenCut team.

## What's in here

This checkout is **two codebases**:

| | Where | What |
|---|---|---|
| **The rewrite** | repo root (`apps/`) | Upstream's ground-up rewrite scaffold — TanStack Start web app, Elysia API, GPUI (Rust) desktop shell, managed by [moon](https://moonrepo.dev) |
| **Classic + my AI agent work** | `classic/` | The previous feature-complete editor, with my additions: an AI editing agent, new GPU effects, and a headless render pipeline |

`classic/` is deliberately **not tracked** by this repo — it's its own git repo, pushed to [debashishthakur/opencut-classic](https://github.com/debashishthakur/opencut-classic). To get a full checkout:

```sh
git clone https://github.com/debashishthakur/opencut.git
cd opencut
git clone https://github.com/debashishthakur/opencut-classic.git classic
```

## The AI editing agent (in `classic/`)

The main thing I've built so far — an agent that edits a video for you from a reference reel:

- **Reference analysis** — decodes a reference reel, detects its shots, and measures its pacing, look, and beat-to-cut coupling
- **Footage profiling** — actually looks at your clips (measured scoring + vision) to pick in-points, instead of guessing
- **Beat sync** — detects the tempo of *your* music and re-instantiates the reference's cutting rhythm on its beat grid
- **No-crop guarantee** — never scales past contain; fills the canvas with a blurred backdrop instead of throwing away pixels
- **GPU color grading** — new WGSL shaders (color grade, vignette, sharpen) working in linear light, plus a shader-generic effects pipeline in Rust
- **Headless rendering** — one HTTP call in, finished MP4 out (`POST /api/agent/render`), no browser tab needed
- **`/create` landing page** — a guided end-to-end flow for testers

[PROGRESS.md](PROGRESS.md) is the detailed architecture audit and goal-by-goal build log — start there.

## Running things

```sh
# The rewrite (repo root)
proto use              # installs moon, bun, rust pinned in .prototools
moon run web:dev       # localhost:5173
moon run api:dev       # localhost:8787
moon run desktop:dev   # native Rust window

# Classic + agent (classic/apps/web)
cd classic && bun install
cd apps/web && bun dev # localhost:3005
```

## License

[MIT](LICENSE) — same as upstream OpenCut.
