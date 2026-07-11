# Ship It: Mission Control — Architecture Design

**Date:** 2026-07-11
**Status:** Proposed (awaiting review)
**Scope:** Whole-prop architecture + the S1→S4 workflow arc. Implementation is decomposed into
milestones (see the end), each of which gets its own plan.

---

## 1. What we're building

A live CI/CD teaching prop for the 2026 DevOps bootcamp. Each learner owns a repo that builds a
small personal **ship microsite**; their `git push` runs a GitHub Actions pipeline whose stages are
**launch phases**. A fully-green run **launches their ship into a shared orbit** on the instructor's
projector — **Mission Control**. A failed test = **ABORT**: the ship stays grounded on its pad.

The prop's job is to give the invisible pipeline a *body*, and to make it **shared** (one orbit for
the whole cohort) and **personal** (every ship is customized). It is a teaching prop, not a product:
we optimize for classroom pedagogy, not engineering completeness.

---

## 2. Resolved architecture decisions

These resolve the open decisions in `PROMPT.md`. They supersede parts of the currently-pinned
`CLAUDE.md` (see §10 for the exact contract changes required).

| Decision | Resolution | Why |
|---|---|---|
| **Ship hosting** | **Serverless.** Static site → **GitHub Pages** via a hand-written workflow, extensible to CF Pages later. | ~60 learners online; no per-learner servers to babysit. Mirrors `devops-bootcamp-app`'s Pages deploy. |
| **Ship rendering** | **Three.js**, using a free **poly.pizza** low-poly rocket `.glb` (vendored, no CDN). | Matches the app's visual bar; the beginner customizes a config file, never the 3D code. |
| **Board hosting** | **Instructor EC2.** One Node process = ws hub **+** serves the Three.js spectator (single origin). | The board holds WebSockets → needs a real server. Same-origin avoids the `wss://` mixed-content problem. Clones the game's `arena-server`. |
| **Board fidelity** | **Rich live ws board** (target). Static "gallery" is a documented lean fallback, not built now. | Per `CLAUDE.md`: build toward rich, keep the lean path graceful. |
| **S4 deploy target** | **The board image**, built by the learner's pipeline, deployed to **their own EC2** (from AWS 2) via **SSM**. | The board is the one artifact that *honestly* needs a server. Ties back to the AWS-2 curriculum. SSM matches the AWS family's SSM>SSH stance. |
| **Who builds the S4 image** | **The learner** authors the `docker build` step (automating the manual local build they already did). | This *is* the CI/CD-4 lesson — automating last week's manual `docker build`. Pulling a prebuilt image would skip it. |
| **Orbit links** | Each orbiting ship links to its live `siteUrl` (the learner's Pages site). | Per the pinned event contract (`siteUrl`). |
| **Testing rigor** | **Drop** Playwright + comprehensive `e2e.sh`. **Keep exactly one** gate: a **config-validation pre-flight check** (not a unit test). | These are fun teaching demos, not production apps. The gate *is* the S2 lesson (exit-code = ABORT); config validation is honest DevOps work, whereas JS unit tests are a developer skill. |

**The one server:** after these decisions, the *only* long-running server in the whole system is
the board. Everything a learner touches (S1–S3) is serverless; S4 is where they stand up a server of
their own.

---

## 3. System architecture

Two components in one monorepo.

```
  INSTRUCTOR (one EC2, security-group :3000 open)
  ┌───────────────────────────────────────────────┐
  │  board  (Node)                                 │
  │   • POST /api/event   ← learner pipelines      │  ← the shared Mission Control (projector)
  │   • WebSocket /       → spectators             │
  │   • serves the Three.js spectator (static)     │
  └───────────────▲───────────────────┬────────────┘
        POST event │ (Bearer token)    │ ws roster broadcast
                   │                   ▼
   ┌───────────────┴───────┐    ┌──────────────────┐
   │ EACH LEARNER'S        │    │ PROJECTOR /       │
   │ GitHub Actions run    │    │ any spectator     │
   │  push → phases → POST │    │ (browser)         │
   └───────────────────────┘    └──────────────────┘

   LEARNER REPO = a FORK of Infratify/shipit-launchpad
     main (forked):  ship/ + ship.config.json + preflight   — NO workflow, NO board/
     the learner AUTHORS .github/workflows/deploy.yml, growing it S1→S4 (the lesson)
     ship/ → GitHub Pages (S1–S3)   ·   board/ (arrives S4) → their own EC2:3000 via SSM (S4)
```

**Data flow (a single green run, S3+):** learner pushes → the workflow runs → at each phase it POSTs
one event (`pad`, `build`, `test`, `clearance`, `liftoff`) to the board with the Bearer token → the
board updates its in-memory roster and broadcasts it over ws → every spectator's Three.js view moves
that ship through countdown → liftoff → orbit. A failed phase POSTs `status: failed` → the ship
visibly aborts and stays on its pad.

---

## 4. Component: `launchpad` — the ship (serverless)

The learner's personal, customizable microsite. Beginner-simple; they edit **one config file**, not
the code.

- **Render:** Three.js scene with a vendored poly.pizza rocket `.glb` (loaded via `GLTFLoader`). The
  config tints the rocket and picks an emblem. A `prefers-reduced-motion` + no-WebGL fallback renders
  a static ship card.
- **Build:** `vite build` → `dist/` (static). Deployed to **GitHub Pages**.
- **Config file — `ship.config.json`** (the thing learners edit; frozen once shipped):

  ```json
  {
    "shipName": "Nebula Runner",
    "color": "#22d3ee",
    "emblem": "comet"
  }
  ```

  - `shipName` — cosmetic label under the rocket. String, 1–24 chars.
  - `color` — hex `#rrggbb`. Tints the rocket + the site accent + the board ship.
  - `emblem` — one of a fixed set: `comet · bolt · star · ring · delta · phoenix` (each a vendored SVG decal).
  - **Identity (`callsign`) is NOT in the config** — it is the learner's GitHub username
    (`${{ github.actor }}`), per the pinned event contract. Unique and un-spoofable; `shipName` is
    just decoration.

- **Config gate — a pre-flight *validation* check (the S2 fitness gate).** Not a unit test. A small
  `scripts/preflight.mjs` validates `ship.config.json` and **exits non-zero on bad config**:
  1. `ship.config.json` parses as JSON.
  2. `color` matches `/^#[0-9a-fA-F]{6}$/`.
  3. `emblem` ∈ the allowed set.
  4. `shipName` is a non-empty string ≤ 24 chars.

  Wired to `npm test`, so learners still see the real CI shape (`npm ci && npm test` → green/red) — but
  there is **no unit-test framework** to learn. Red when a learner typos the config → the pipeline
  ABORTS. That *is* the entire S2 lesson: edit config → `npm test` → green/red. The transferable idea
  is the **exit-code gate** (DevOps), not test authoring (developer). No `vitest` dependency.

- **No container.** The ship is static; it never has a Dockerfile. (The container lesson lives on the
  board — see §5.)

---

## 5. Component: `board` — Mission Control (the one server)

Clones the game's `arena-server` shape: an `http` server + `ws` sharing one port, an in-memory roster
with a `dirty`-flag broadcast tick, tiny JSON message helpers. Differences from the game:

- **Ingest is HTTP POST, not ws.** Events arrive from workflows at `POST /api/event`; only spectators
  hold ws connections.
- **Auth.** `POST /api/event` requires `Authorization: Bearer $SHIPIT_TOKEN`. Unauthenticated events
  are rejected in prod mode (the "no clearance → can't report" lesson).
- **Spectator is Three.js** (the showpiece), served static from the same process. Blueprint aesthetic,
  projector-legible, `prefers-reduced-motion` + no-WebGL fallback (a plain roster list).

**Roster / orbit model.** Keyed by `callsign` (GitHub username). Each entry holds
`{ callsign, stage, status, color, version?, siteUrl? }`. The spectator renders each ship on its pad,
in-flight, or in orbit according to `stage`/`status`. Ephemeral — no persistence beyond the current
session (arena pattern).

**Two roles for one image (`ghcr.io/infratify/shipit-board`):**
1. **Shared Mission Control** — the instructor runs the canonical instance on instructor EC2; it's the
   projected orbit the whole cohort launches into. Runs all month.
2. **The learner's S4 build+deploy target** — in S4 each learner's pipeline builds *this same source*
   and deploys their *own* instance to *their own* EC2. A personal victory lap: "run your own Mission
   Control." Their instance is standalone (shows whatever posts to it).

**Publish.** Multi-arch (`amd64`/`arm64`) GHCR image on `v*` tag; public before class.

---

## 6. PINNED event contract (unchanged)

The one integration point. Kept stable from `CLAUDE.md`:

```
POST  $BOARD_URL/api/event
Authorization: Bearer $SHIPIT_TOKEN
Content-Type: application/json

{
  "callsign": "octocat",       // GitHub username (${{ github.actor }})
  "stage":    "build",         // pad | build | test | clearance | liftoff
  "status":   "passed",        // running | passed | failed | aborted | shipped
  "color":    "#22d3ee",       // from ship.config.json
  "version":  "v3",            // optional; image/site tag
  "siteUrl":  "https://…"      // optional; the live Pages site to link from orbit
}
```

- `$BOARD_URL` — public repo/environment **variable**.
- `$SHIPIT_TOKEN` — the **secret** taught in S3. The POST step is *added to the workflow in S3*; from
  then on every run reports its full phase sequence. Before S3 the learner's payoff is the Actions run
  + the live Pages URL, not the board.

---

## 7. The learner repo + the 4-session workflow arc

**Distribution: learners FORK, they don't get a template.** The learner-facing starter is a dedicated
repo — **`Infratify/shipit-launchpad`** — that each learner **forks**. A fork (not "Use this template")
is deliberate: it keeps the upstream link, so learners can **sync** instructor fixes mid-cohort and
`git diff` their work against the solution branches. This monorepo (`devops-bootcamp-shipit`) stays the
*source* that produces `shipit-launchpad` (the `board/` and ship sources live here; a release step
syncs them out).

**Payload vs pipeline — the load-bearing split.** The learner is *given* the app (payload) and
*authors* the pipeline (the lesson):

- **Payload (shipped on `main`):** `ship/`, `ship.config.json`, `scripts/preflight.mjs`, `package.json`
  (with `build`/`test`), README. This is what a developer hands a DevOps engineer.
- **Pipeline (authored by the learner):** `.github/workflows/deploy.yml` — **absent from `main`** —
  written from scratch, growing one job per session. Putting them in the app-owner's-code-exists,
  you-write-the-automation seat is the whole point.

**Branch model on `shipit-launchpad`:**

| Branch | Contains | Role |
|---|---|---|
| **`main`** | ship payload + config + preflight — **no workflow, no `board/`** | what learners **fork & build on**; syncable for fixes |
| `cicd1` | `main` + S1 `deploy.yml` (Pages) | answer key / catch-up for S1 |
| `cicd2` | + S2 test-gate job | answer key S2 |
| `cicd3` | + S3 secret + board POST step | answer key S3 |
| `cicd4` | + S4 build+SSM job **+ `board/`** | answer key S4; also where the dashboard payload enters |

- The workflow a learner writes lives only on **their** `main`; the `cicd*` branches are reference-only
  (`git diff upstream/cicd2`), or a reset target if hopelessly behind. Slides lift YAML straight from
  these branches — so the old `docs/reference-workflows/*.yml` idea is **dropped**.
- **`board/` arrives just-in-time (S4)** off the `cicd4` branch: `git checkout upstream/cicd4 -- board/`.
  So S1–S3 stay focused on the ship; the payload grows alongside the pipeline. `board/` is a black box
  they *build and ship*, never edit — realistic (you deploy services you didn't write).

**Why the payload-only `main` is the linchpin** — it protects both the lesson and the fork mechanics:
- Learners still author the pipeline from nothing.
- No fork "enable inherited workflows" security prompt — `main` has no workflow, so their own pushed
  workflow just runs as theirs. (A fresh fork still needs Actions enabled once — a one-time click.)
- **Syncs stay conflict-free** *as long as upstream `main` never re-touches `ship.config.json` and never
  gains a workflow after release.* Learners only ever *add* `deploy.yml` (absent upstream) and *edit*
  `ship.config.json` (frozen upstream), so a sync merges cleanly. **This discipline rule is
  load-bearing** — it is what keeps beginners out of merge conflicts.

**The lean arc — one headline concept per session.** Secondary items are optional "stretch," never
required hands-on.

| Session | The one concept | Automates | What you see | Stretch (optional) |
|---|---|---|---|---|
| **S1** | a pipeline deploys on `push` | copying files to a host by hand | **pad lights up** — ship site live on Pages | — |
| **S2** | a test **gate** can block you | eyeballing whether it worked | **systems check** — green = go · red = **ABORT** | matrix (2 engines) · artifact upload |
| **S3** | **secrets** let your ship talk to Mission Control | manually updating a status | **first contact** — token comes online, ship appears live on the shared board | environment + manual approval ("Mission Control gives GO") |
| **S4** | your pipeline **builds a container and runs it on your server** | `docker build` + deploy on your laptop | **LIFTOFF** — your own Mission Control on your EC2 | image tags + rollback demo |

**Token model — two tokens, both meaningful:**
- `GITHUB_TOKEN` (built-in) — pushes the board image to the learner's GHCR in S4 (no manual secret).
- `$SHIPIT_TOKEN` (S3 secret) — authorizes reporting to the *shared* Mission Control.

**Answer keys = the `cicd*` branches** (see the branch table above). Slides quote their `deploy.yml`
verbatim; a stuck learner diffs against or resets to them. No separate `docs/reference-workflows/`.

**Per-session commands (the "kelas-taip-bersama" lines).** Representative; finalized during the
starter-repo milestone:
- **Setup (before S1):** **Fork** `Infratify/shipit-launchpad` → enable Actions on the fork (one click).
- **S1:** edit `ship.config.json`; hand-write `.github/workflows/deploy.yml` (type-along) → `git commit -am "my ship" && git push` → watch Actions → open the Pages URL.
- **S2:** add the test-gate job (type-along); typo the `color` → `git push` → watch it go red (**ABORT**) → fix → green.
- **S3:** add the board-POST step; `gh secret set SHIPIT_TOKEN` + set the `BOARD_URL` variable → `git push` → your ship appears on the shared board.
- **S4:** `git checkout upstream/cicd4 -- board/` to pull the dashboard; add the build+deploy job → `git push` → the pipeline builds `board/`, pushes to your GHCR, SSM-deploys to your EC2 → open `your-ec2:3000`.
- **Catch-up (any session):** `git diff upstream/cicdN` to compare, or reset to it if lost. **Sync fork** to pull instructor fixes.

---

## 8. Conventions

- Node 20, ESM. Fail loud, no swallowed errors.
- Self-contained: **no CDN** — vendor/bundle everything (including the poly.pizza `.glb` and Three.js).
- Theme-aware. WebGL + `prefers-reduced-motion` fallbacks on anything animated (both components).
- Gate: **one** config-validation pre-flight check on the ship (`npm test` → `node scripts/preflight.mjs`),
  **not** a unit-test framework. No `vitest`, no Playwright, no comprehensive e2e. A light
  `scripts/smoke.sh` may `curl` the board to prove the POST→ws loop, but it is not a gate.
- The board is projector-legible; the ship is beginner-customizable. Approachable beats high-fidelity
  on the ship; spectacle lives on the board.

---

## 9. Non-goals (explicitly dropped)

- A ship container / `shipit-launchpad` image — the ship is static-only.
- Unit-testing frameworks (`vitest`) — the S2 gate is a config *validator*, not unit tests.
- Playwright e2e and a full-loop `scripts/e2e.sh` proof.
- Matrix builds, artifact upload, environments/approval, image tags, rollback **as required content** —
  all demoted to optional stretch.
- Persistence on the board — roster is ephemeral per cohort session.
- The static "gallery of shipped ships" fallback board — documented as a lean path, not built now.

---

## 10. Required changes to `CLAUDE.md` pinned contracts

To be applied **after this spec is approved** (they change the durable source of truth):

1. **Components table:** remove `shipit-launchpad` (`:8080`) as an image — the ship is serverless/static.
   (The freed `shipit-launchpad` name is reused for the forkable learner *repo* — see item 7.)
   The board stays `ghcr.io/infratify/shipit-board` (`:3000`) and gains its dual role.
2. **Session arc:** replace the dense S2/S4 rows with the lean one-concept-per-session arc (§7);
   record matrix/artifact/environment/approval/tags/rollback as stretch.
3. **S4 mechanism:** the S4 deploy pushes the **board** image to the learner's GHCR and runs it on the
   learner's own EC2 via SSM (was: launchpad image to EC2).
4. **Learner-facing contract:** fill the placeholders — `ship.config.json` schema (§4), the config
   validation gate (§4), per-session commands (§7).
5. Note `callsign` = `${{ github.actor }}`; `shipName` is the cosmetic config field.
6. **Conventions — tests line:** replace "Tests: `vitest` unit + `playwright` e2e" with "one
   config-validation pre-flight check on the ship (exit-code gate); no unit-test framework, no
   Playwright." (The S2 gate teaches the gate, not test authoring — a DevOps vs developer distinction.)
7. **Distribution model:** learners **fork** `Infratify/shipit-launchpad` (payload-only `main` +
   `cicd1..4` answer-key branches); they *author* the workflow, it is not shipped. Record the
   payload-only-`main` discipline rule (§7) so slides and fixes respect it.

---

## 11. Build milestones (each → its own plan)

1. **`launchpad` (ship) MVP** — Three.js rocket + poly.pizza `.glb` + `ship.config.json` + the config
   pre-flight validation gate (`scripts/preflight.mjs`) + static `vite build`. Runs locally, builds to
   `dist/`, reduced-motion/no-WebGL fallback.
2. **`board` MVP** — ws hub (arena-server pattern) + `POST /api/event` (Bearer auth) + Three.js
   spectator rendering pads/roster; serves static; `node:20-alpine` Dockerfile.
3. **Wire the contract end-to-end** — a `curl` from a fake workflow moves a ship pad → liftoff → orbit
   on the board live (`scripts/smoke.sh`).
4. **Mission Control polish** — countdown, liftoff, orbit; blueprint style; projector-legible;
   fallbacks solid.
5. **Learner starter repo `shipit-launchpad`** — produce the forkable repo from this monorepo:
   payload-only `main` (ship + config + preflight, **no workflow, no `board/`**) + `cicd1..4`
   answer-key branches (each = full correct state; `board/` enters at `cicd4`). Author the four
   `deploy.yml` end-states, finalize per-session commands, verify a fresh fork syncs cleanly.
6. **CI publish** — multi-arch GHCR publish workflow for `shipit-board` (the only image); the release
   step that syncs `board/` + ship sources from this monorepo out to `shipit-launchpad`; images public
   before class.

Milestones run in sequence; each becomes a `writing-plans` document before implementation.
