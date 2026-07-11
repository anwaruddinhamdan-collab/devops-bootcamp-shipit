# launchpad — the learner's ship

The repo every learner pushes to. A small **personal ship microsite** they customize (callsign,
colour, emblem), and the thing their pipeline builds, tests, and ships across all four CI/CD
sessions. Their **one workflow grows on this repo** — don't split it per session.

**Keep it beginner-simple.** Light Three.js or 2D canvas — approachable to customize beats
fidelity here. It must:

- build to static (`vite build` → `dist/`) for the **Pages** deploy in CI/CD 1;
- carry a **config test** (`vitest`) that is the CI/CD 2 "fitness gate" (red = ABORT);
- have a small **playwright** smoke;
- ship its **own multi-stage Dockerfile** (`preview` on `:8080`) for the CI/CD 4 GHCR + EC2 deploy.

Each stage of the workflow POSTs to Mission Control — see the pinned contract in the repo-root
`CLAUDE.md`. Config the learner edits (freeze once decided): e.g. `ship.config.json` →
`{ callsign, color, emblem }`.

Image: `ghcr.io/infratify/shipit-launchpad`, port `8080`. Starter `package.json` is a stub.
