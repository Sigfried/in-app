# in-app monorepo

The plan for what gets built here is [docs/PLAN.md](docs/PLAN.md).

## Context

- pnpm + Nx monorepo for shared packages under the `@sigfried/` npm scope,
  named `@sigfried/in-app-*`.
- Existing apps (dmvd, vs-hub) are brought in temporarily **with full git
  history**, so they can be refactored alongside the packages. When stable,
  each is split back out to its original GitHub repo. Everything below serves
  that round trip.
- **The live sites are frozen** while their apps are here. vs-hub (GitHub
  Actions) and dmvd (`npm run deploy`, which runs `gh-pages`) are not
  redeployed until each is split back out.

## Layout

- `packages/*` — shared packages, eventually published to npm.
- `apps/*` — absorbed apps, one directory per original repo.
- [pnpm-workspace.yaml](pnpm-workspace.yaml) lists the workspaces. One root
  `pnpm-lock.yaml`. **vs-hub's app is in `frontend/`**, not at its repo root,
  so its workspace entry is `apps/vs-hub/frontend`.

## Bringing an app in

- `git subtree add --prefix=apps/<name> <repo-url> main` **without**
  `--squash`.
- Do not rewrite, rebase, or squash any commits under `apps/<name>` afterward.
- Delete the app's own lockfile after absorbing; the root lockfile replaces it.

## Nx

- Package-based mode: tasks come from each package's `package.json` scripts.
- Do not convert projects to integrated style, and do not let generators add
  `tsconfig` `paths` aliases, root-level `project.json` conventions, or shared
  config that an app would need to function.

## Rules that keep apps extractable

- Apps import shared packages **only by package name**
  (`@sigfried/in-app-tour`), declared in the app's `package.json` as
  `"workspace:*"`. No relative imports that leave the app's directory, no
  tsconfig path aliases into `packages/`.
- Each app's tsconfig, ESLint, Prettier and Vite configs stay self-contained
  inside `apps/<name>/`. They do not extend root-level configs.
- Every dependency an app uses is declared in that app's own `package.json`,
  even if it is already installed at the root.
- Shared packages declare React (and other framework libraries) as
  `peerDependencies`, not `dependencies`.

## Hard limits and reporting

- Do NOT push to, open PRs against, or change settings on the original app
  repos.
- Do NOT run `git subtree push` or `git subtree split` to extract an app unless
  Siggie explicitly asks for it in that session.
- Do NOT publish packages to npm unless Siggie explicitly asks.
- After any change, say what was actually run to verify it (install, build,
  typecheck, tests, dev server) and what was not. Report failures, warnings and
  skipped steps plainly; "builds, but I didn't run the app" is the kind of
  report wanted.

## Known traps

- **pnpm's `node_modules` is symlinks.** A symlinked `node_modules` has broken
  vitest under the Claude Code sandbox before (dmvd worktrees): the sandbox's
  allowed paths are matched against *real* paths. Expect to resolve real paths
  (`pwd -P`) and adjust the allowlist.
- **Node 24.** Pinned in [.nvmrc](.nvmrc) (24.2.0). On node 26, 41 of dmvd's
  tests fail (jsdom `localStorage`). vs-hub's deploy workflow also uses 24.
- **vs-hub brings its data.** `apps/vs-hub/data/` is ~45 MB of committed
  Parquet.
- **`pnpm install` runs inside the sandbox** because
  [.claude/settings.json](.claude/settings.json) allows writes to
  `~/Library/pnpm` and `~/Library/Caches/pnpm`, and the `registry.npmjs.org`
  domain. It must be all of `~/Library/pnpm`, not just its `store/`: pnpm
  tests the parent, and when it can't write there it silently picks an
  in-repo `.pnpm-store`, then wants to delete `node_modules` to reinstall
  against it.
- **`~/.npmrc` is hidden from the sandbox**, so the npm auth token in it can't
  be used: that is what makes allowing the registry domain safe. Every pnpm
  command warns `EPERM ... .npmrc`; that warning is expected.
- **nx works in the sandbox** through the socket allowances for `/tmp/.nx` in
  the same settings file. Run tasks through nx (`pnpm nx run-many -t build`),
  not around it.
- **No work in the original clones while absorbed.** A commit in
  `~/github-repos/dynamic-model-var-docs` or `personal/vs-hub` would diverge
  from `apps/*`. The dmvd dev server moves too: run it from `apps/dmvd`.
- **dmvd's Claude memory doesn't carry over.** It's keyed to the old path. The
  gotchas that matter are in [apps/dmvd/docs/CLAUDE.md](apps/dmvd/docs/CLAUDE.md)
  and this section.
