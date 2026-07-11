# Kickoff prompt — "Ship It: Mission Control"

> **Open a fresh Claude session in this repo and paste (or point it at) this file.**
> It is the brief. Read `CLAUDE.md` next for the pinned contracts, then brainstorm →
> spec → plan before writing code. Do NOT start coding from this file alone.

---

You are building a **live CI/CD teaching prop** for the 2026 DevOps bootcamp — a sibling to
two existing props:

- `Infratify/devops-bootcamp-app` — a **Three.js scrollytelling** explainer that scroll-builds a
  multi-stage Dockerfile. *This is the visual bar to match.*
- `Infratify/devops-bootcamp-game` — a **PixiJS multiplayer avatar arena**; students run
  containers on their laptop and their character walks into a shared room on a projector.
  *This is the "shared, live, personal" interaction bar to match.*

Those two are Docker-flavoured. **This prop is CI/CD-native:** it makes an invisible pipeline
**visible, shared, and personal**, the way the arena made networking *felt*.

## The concept — "Ship It"

A rocket-launch pipeline. Every learner owns a repo that builds a small personal **ship
microsite**. Their `git push` triggers a GitHub Actions pipeline; **each pipeline stage is a
launch phase**; a fully-green run **launches their ship into a shared orbit** shown on the
projector — **"Mission Control."** A failed test = **ABORT**: their ship is visibly grounded on
its pad until they fix it. ~60 learners, fully online (Mission Control is shown via instructor
screen-share).

The magic moment (retargeted from the arena): **push → watch your pipeline flow through the
launch phases live on the big screen → green → your ship lifts off and joins a shared orbit you
can click.**

## Why — the pedagogy this serves (build in service of this)

The prop rides a **4-session arc**. The learner's **one** workflow file grows each session; the
ship gets one phase closer to orbit. Felt-need spine: each session automates a step they did by
hand the week before.

| Session | Launch phase | CI/CD content taught |
|---|---|---|
| **CI/CD 1** (14 Jul) | Pad goes live on **Pages** | first workflow · trigger (push) · job/step/runner |
| **CI/CD 2** (16 Jul) | Fuelling + systems check | **build** · **test gate** (red = ABORT) · matrix (2 engines) · artifact |
| **CI/CD 3** (21 Jul) | Launch clearance | **secrets** · **environment** · **manual approval** (Mission Control gives GO) |
| **CI/CD 4** (23 Jul) | **LIFTOFF → orbit** | image → **GHCR** → deploy to EC2 (`:8080`) · tags · **rollback** |

## Architecture — one monorepo, two parts

Mirror the game's `server/ + avatar/` split:

- **`board/` — Mission Control.** A small Node ws hub (roster + broadcast; clone the
  `arena-server` pattern) **+ a Three.js spectator/projector view**. Learner pipelines POST
  events to it; spectators watch launch phases and orbit live. **This is the showpiece** — full
  Three.js/animated/blueprint polish, like `devops-bootcamp-app`.
- **`launchpad/` — the learner template.** A small **buildable / testable / containerizable**
  ship microsite. Customized via a config file (callsign, colour, an emblem/thruster) → every
  learner's ship is unique. `vite build` → `dist/` (Pages, S1); a **vitest test on the config**
  is the "fitness gate" (S2); a **playwright** smoke; its **own multi-stage Dockerfile** (GHCR +
  EC2 `:8080`, S4). Keep it **beginner-simple** — light Three.js or 2D canvas; approachable to
  customize is worth more than fidelity here.

## Milestones (suggested order)

1. **launchpad MVP** — customizable ship microsite + the config `vitest` gate + Dockerfile; runs locally, builds to static.
2. **board MVP** — ws hub + spectator that renders a roster of ships and their current launch phase.
3. **Wire the event contract end-to-end** — a `curl` from a fake workflow updates a ship on the board live.
4. **Mission Control polish pass** — Three.js countdown, liftoff, orbit; blueprint style; projector-legible; reduced-motion + no-WebGL fallback.
5. **Ship the reference learner workflows, staged S1→S4** — so the bootcamp slides can quote them verbatim (like the arena's pinned commands).
6. **CI + e2e** — multi-arch GHCR publish workflow for `shipit-board` + `shipit-launchpad`; an `scripts/e2e.sh` proving the full loop.

## Open decisions — resolve early, record in CLAUDE.md

- **Board hosting:** instructor EC2 (like `arena-server`) vs Cloudflare (Pages/Workers/DO).
- **Board fidelity:** live ws board (rich, target) vs static "gallery of shipped ships" (lean fallback). Build toward rich; keep the lean path graceful.
- **launchpad rendering:** light Three.js ship vs 2D canvas — pick the one a beginner can safely customize.
- **S4 deploy mechanism:** SSM (leaning this — matches the AWS family's SSM>SSH stance) vs self-hosted runner vs SSH, deploying the launchpad container to the learner's own EC2 (from AWS 2).
- **Orbit links:** each orbiting ship links to that learner's live deployed `:8080` site.

## Constraints (match the sibling repos)

- Node 20, ESM. Fail loud, no swallowed errors.
- Self-contained (no CDN — vendor or bundle). Theme-aware. WebGL + `prefers-reduced-motion` fallbacks on anything animated.
- Tests: `vitest` unit + `playwright` e2e (like `devops-bootcamp-app`).
- Multi-arch (`amd64`/`arm64`) GHCR publish on a `v*` tag; images public before class.
- **Pin the learner-facing contract** (commands, filenames, the event payload) in `CLAUDE.md` — the bootcamp slides will depend on it being stable, exactly like the arena contract.

## Start here

Read `CLAUDE.md`. Then propose a build plan (brainstorm → spec → plan). Confirm the open
decisions above with the human before committing to an architecture.
