# orca-dag: AZM fork

_Created: 2026-10-09_

Josh's fork of [ZinkLu/Orca-Orchestration](https://github.com/ZinkLu/Orca-Orchestration):
the `orca-dag` skill plus the DAG viewer. Upstream's working guidance is in AGENTS.md and is
imported here:

@AGENTS.md

**Why this file isn't the upstream symlink:** upstream ships `CLAUDE.md` as a symlink to
`AGENTS.md`. This clone has `core.symlinks=false` (the Windows default), so git checked it out
as a 9-byte text file containing `AGENTS.md`, and Claude never saw the guidance. Exclude this
file from any PR back to upstream.

## Remotes and branches
- `origin` = `jyoder35/Orca-Orchestration` (the fork). `upstream` = `ZinkLu/Orca-Orchestration`.
- `main` tracks upstream. AZM work lives on `azm/skill-field-lessons`.
- **The branch checked out in this folder is the skill every agent reads** (see below). Do
  risky or long edits in an Orca worktree and merge them back. Don't park this folder on an
  experimental branch.

## The skill: one live copy for every agent
`skill/SKILL.md` here is the only copy. These are junctions to this repo's `skill\` folder
(set up 2026-10-01):

| Agent | Folder |
|---|---|
| Claude Code | `AI\.claude\skills\orca-dag` (`~\.claude` is itself a junction to `AI\.claude`) |
| Codex | `~\.codex\skills\orca-dag`, plus the shared `~\.agents` copy |
| Cursor | `AI\.cursor\skills\orca-dag` (`~\.cursor` is a junction to `AI\.cursor`) |
| OpenCode | `~\.config\opencode\skills\orca-dag` |
| Shared (Amp, Cline, etc.) | `~\.agents\skills\orca-dag` |

- Saving `SKILL.md` updates every agent at once. There's no install step.
- Agents load skills when a session starts, so open a fresh agent tab to pick up an edit.
- The viewer's installer (`server/src/skill.ts`) skips linked folders. Node reports these
  junctions as symlinks, which is what makes the skip work. Never replace a junction with a
  real folder, and don't set `ORCA_DAG_NO_SKILL`.
- If a new agent folder appears later (e.g. `~\.gemini`), the installer writes a plain copy
  there from `dist-npm/skill/`. Replace that copy with a junction.
- Run `node scripts/check-skill.mjs` after editing the frontmatter.
- Skill lessons stay generic. Describe Orca's behaviour and date it, with no AZM business
  details, so they could go upstream.

## Running the viewer from this clone
- The global `orca-dag` command runs this clone. `%APPDATA%\npm\node_modules\orca-dag` is a
  junction to `dist-npm/` (set with `npm install -g ./dist-npm` on 2026-10-09).
- **After changing viewer code:** run `npm run build:npm`, then restart any running viewer.
- **To use it:** open a terminal in an Orca-managed worktree of the project whose Runs you
  want, run `orca-dag`, and keep that tab open. Use `orca-dag`, not `npx orca-dag`; npx may
  fetch the stock package from npm instead.
- **UI dev loop:** `npm run dev` in this folder starts the server on :8787 and Vite on :5173
  with hot reload. Open :5173. Stop any other viewer on :8787 first.
- **Back to stock:** `npm install -g orca-dag`.
- **Windows build fix (fork only):** `scripts/build-npm.mjs` and `build-binary.mjs` failed on
  Windows (`spawnSync npm ENOENT`, then the esbuild shim). npm now runs through a shell on
  win32, and esbuild is called as `esbuild.exe`. This is worth offering upstream.

## Key file locations
| What | Path |
|---|---|
| Skill (live for all agents) | `skill/SKILL.md` |
| Upstream agent guidance | `AGENTS.md` |
| Viewer UI | `web/src/` (`App.tsx`, `layout.ts`, `harness.ts`, `DoodleSelect.tsx`) |
| Viewer server and scheduler | `server/src/` (`index.ts`, `coordinator.ts`, `orca.ts`, `skill.ts`) |
| Build and release scripts | `scripts/` |
| Global install target (generated, gitignored) | `dist-npm/` |
| Unshipped proposals and mockups (e.g. UI redesign) | `planning/proposals/`, `planning/mockups/` |

## Fork rules
- Match upstream's style in upstream files: English, with rich "why" comments on anything
  that touches Orca.
- Keep fork-only changes (this file, AZM wording) separate from changes worth upstreaming.
- **Never run `npm run release`.** It tags and pushes, and the release workflow then publishes
  to npm.
- Viewer redesign: put the proposals and mockups under `planning/` and get Josh's approval
  before changing `web/src/`.
