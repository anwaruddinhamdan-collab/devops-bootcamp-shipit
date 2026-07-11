# CLAUDE.md — Ship It: Mission Control

Design rationale and the **pinned contracts** for this CI/CD teaching prop. Read `PROMPT.md`
first for the build brief. This file is the durable source of truth once the build starts —
keep the contracts here stable, because the bootcamp slides quote them.

## What this is

A live CI/CD teaching prop for the 2026 DevOps bootcamp. Learners each ship a personal **ship
microsite** through a GitHub Actions pipeline; a green run **launches their ship into a shared
orbit** on the projector ("Mission Control"). It makes the invisible pipeline visible, shared,
and personal — the CI/CD counterpart to the arena prop.

Distinct from its siblings (do not blur them):
- `devops-bootcamp-app` — Three.js Docker-layers scrollytelling. **Match its visual bar.**
- `devops-bootcamp-game` — PixiJS avatar arena. **Match its shared-live-personal interaction.**

## Why it's shaped this way

- **The pipeline is the abstract thing.** CI/CD is YAML + green checks + logs — invisible. The
  prop's whole job is to give the pipeline a *body*: launch phases you watch, a ship that either
  reaches orbit or aborts on its pad.
- **One repo, one growing workflow.** Each learner keeps ONE launchpad repo across all four
  sessions; the workflow file *grows* (Pages → build/test → secrets/approval → GHCR/deploy).
  Four throwaway projects would kill the "watch it mature" payoff.
- **Personal identity in a shared world.** Every ship is customized (callsign, colour, emblem),
  so the shared orbit is 60 distinct ships, not 60 identical ones — the reason the arena landed.
- **Felt-need spine.** Each session automates last week's manual step; the ship visibly gets
  closer to orbit as the pipeline does more of the work.

## Components (target)

| Component | Built from | Image | Port |
|---|---|---|---|
| `shipit-board` — Mission Control (ws hub + Three.js spectator) | `board/` | `ghcr.io/infratify/shipit-board` | 3000 |
| `shipit-launchpad` — the learner ship microsite template | `launchpad/` | `ghcr.io/infratify/shipit-launchpad` | 8080 |

## PINNED — pipeline ↔ board event contract

The one integration point. Keep it stable; slides and the reference workflows depend on it.

- **Identity** = the learner's GitHub username (`${{ github.actor }}`), used as `callsign`.
- **Config** the board needs comes from the learner's `launchpad` config (colour, emblem).
- **Transport:** each workflow stage POSTs one event to the board.

```
POST  $BOARD_URL/api/event
Authorization: Bearer $SHIPIT_TOKEN
Content-Type: application/json

{
  "callsign": "octocat",          // GitHub username
  "stage":    "build",            // pad | build | test | clearance | liftoff
  "status":   "passed",           // running | passed | failed | aborted | shipped
  "color":    "#22d3ee",          // from the learner's ship config
  "version":  "v3",               // optional; image/site tag (for rollback demo)
  "siteUrl":  "https://…"         // optional; the live deployed site to link from orbit
}
```

- `$BOARD_URL` is a **public** repo/environment **variable**.
- `$SHIPIT_TOKEN` is the **secret** taught in CI/CD 3 — a ship with no/late token can't report to
  Mission Control (the "unauthorized" lesson). Do NOT accept unauthenticated events in prod mode.
- The board keeps an ephemeral roster and broadcasts it to spectators over WebSocket (arena
  pattern). No persistence required beyond the current cohort's session.

## PINNED — learner-facing contract (fill in as the build firms up)

The slides will quote these verbatim, so freeze them once decided. Placeholders for the build
session to make concrete:

- Launchpad **config file** learners edit (name + shape). e.g. `ship.config.json` → `{ callsign, color, emblem }`.
- The **config test** that is the S2 fitness gate (what it asserts).
- The **reference workflow** at each session's end state (S1 → S4), stored so slides can lift them.
- Exact **commands** learners run each session (the "kelas-taip-bersama" lines).

## Conventions

- Node 20, ESM. Fail loud. No CDN (vendor/bundle). Theme-aware. WebGL + reduced-motion fallbacks.
- Tests: `vitest` unit + `playwright` e2e. Multi-arch GHCR publish on `v*` tag; images public before class.
- `launchpad` stays beginner-simple; `board` carries the Three.js spectacle.

## Bootcamp integration (context; the arc itself lives in the slides repo)

`~/repo/slides-devops-bootcamp` → `outlines/2026/cicd1..4.md` + `slides/2026/cicd1..4/`. The
`$SHIPIT_TOKEN` is the CI/CD 3 secret; the S4 deploy pushes the `launchpad` image to GHCR and
runs it on the learner's own EC2 (from AWS 2) via SSM. Board runs on instructor infra.
