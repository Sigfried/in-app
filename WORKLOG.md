# WORKLOG

## 2026-09-29 — monorepo created (PLAN §4.8, first session)

- Absorbed both apps with `git subtree add` from the local clones (no
  `--squash`): dmvd at `c92a47b` (1 commit ahead of its origin; that commit is
  the plan seed), vs-hub at `e73055b`. `subtree add` makes its own merge
  commits, so those two went in without a separate ask.
- Siggie's sandbox question: we kept the sandbox on. The only planned steps it
  blocks are `pnpm install` / `pnpm add` (global store + registry), which
  Siggie runs. The outward-facing risks it guards are `npm publish` (a live
  npm token exists) and dmvd's `npm run deploy` (gh-pages).
- Root `engines.node` is `>=24 <25`, not just 24.2.0: `.nvmrc` carries the exact
  pin, and `engines` is there to reject node 26 (41 dmvd test failures).
- §4.1–4.7 of the plan moved to [CLAUDE.md](CLAUDE.md), with the frozen-site
  decision from §1 added there. The plan's §4 now keeps only the decision and
  the remaining setup steps. The plan's links were relative to dmvd's `docs/`
  and were repathed to `../apps/dmvd/...`.
- `apps/dmvd/docs/HELP_PACKAGE_PLAN.md` is now a pointer, not deleted, because
  dmvd's README and TASKS link to it. Its link (`../../../docs/PLAN.md`) will
  break when dmvd is split back out.
- Expected risk for the install/verify step: vs-hub has only ever been installed
  with npm's flat `node_modules`. pnpm is strict, so any import of a package
  vs-hub doesn't declare (a phantom dependency) will fail its build. That
  would need fixing by declaring the dependency in vs-hub's `package.json`,
  per the extractability rules.
