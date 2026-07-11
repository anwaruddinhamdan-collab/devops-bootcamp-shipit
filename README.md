# devops-bootcamp-shipit

**"Ship It: Mission Control"** — a live CI/CD teaching prop for the 2026 DevOps bootcamp.

Every learner ships a small personal **ship microsite** through a GitHub Actions pipeline. Each
pipeline stage is a **launch phase**; a fully-green run **launches their ship into a shared orbit**
on the projector. A failed test = **ABORT** — grounded on the pad until fixed. It makes an
invisible pipeline visible, shared, and personal.

Sibling props: `devops-bootcamp-app` (Three.js Docker scrollytelling — the visual bar) and
`devops-bootcamp-game` (PixiJS avatar arena — the shared-live-personal bar). This one is
CI/CD-native.

## Status

🚧 **Scaffold only.** This repo currently holds the brief, not the build.

- **`PROMPT.md`** — kickoff brief. Open a fresh Claude session here and start from it.
- **`CLAUDE.md`** — design rationale + the pinned pipeline↔board contract.

## Layout (target)

```
board/       Mission Control — ws hub + Three.js projector spectator   → ghcr.io/infratify/shipit-board  (:3000)
launchpad/   the learner ship microsite template (build/test/Dockerfile) → ghcr.io/infratify/shipit-launchpad (:8080)
scripts/     e2e.sh — full loop proof (to be written)
docs/        specs + plans
```

## The 4-session arc it serves

| Session | Launch phase | Teaches |
|---|---|---|
| CI/CD 1 | Pad live on Pages | first workflow · trigger · job/step/runner |
| CI/CD 2 | Fuelling + systems check | build · test gate (red = ABORT) · matrix · artifact |
| CI/CD 3 | Launch clearance | secrets · environment · manual approval |
| CI/CD 4 | LIFTOFF → orbit | image → GHCR → deploy to EC2 · tags · rollback |

## Next step

Read `PROMPT.md`, then brainstorm → spec → plan before writing code.
