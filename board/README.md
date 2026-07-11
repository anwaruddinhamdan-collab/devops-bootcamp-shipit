# board — Mission Control

The projector showpiece. A small Node WebSocket hub that receives pipeline events from every
learner's workflow and broadcasts a live roster to a **Three.js spectator view** — launch pads,
countdowns, liftoffs, and a shared orbit of shipped ships.

**Match the visual bar of `devops-bootcamp-app`:** Three.js, animated, blueprint aesthetic,
self-contained (no CDN), WebGL + `prefers-reduced-motion` fallbacks, projector-legible.

- Event ingest + auth: see the **pinned contract** in the repo-root `CLAUDE.md`.
- Pattern to clone: `devops-bootcamp-game`'s `server/` (ws roster hub + `public/` spectator).
- Image: `ghcr.io/infratify/shipit-board`, port `3000`.

Starter `package.json` is a stub — finalize deps/scripts during the build.
