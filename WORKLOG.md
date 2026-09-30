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

## 2026-09-29/30 — install, verify, sandbox permissions

- Verify results: dmvd vitest 821 passed / 3 skipped (no node-26 failures, no
  symlink trap); vs-hub build fine. dmvd typecheck failed on an undeclared
  `unified` (a type import in `markdownParts.tsx`) — the phantom-dependency
  risk above hit dmvd, not vs-hub. Declared as a devDependency, `^11.0.5`,
  the version both lockfiles already resolved.
- Siggie's shell runs pnpm under node 26 even with nvm on 24 (`which -a pnpm`
  shows only nvm's pnpm, whose shebang is `env node`, so node 26 must come
  first on PATH in their interactive shell at the time pnpm runs). Harmless
  for installs; unresolved. Claude's shell gets 24.
- **nx socket.** nx needs a Unix socket under `/tmp/.nx`. I first proposed
  running tasks with `pnpm --filter` to dodge it; Siggie objected ("i don't
  want you doing weird stuff to work around sandbox restrictions") and asked
  for the permission instead. Ran `nx configure-ai-agents --agents claude`
  (Siggie, from their shell), which wrote the sandbox allowances to
  `.claude/settings.json`, ignore lines to `.gitignore`, a third-party plugin
  (`nx@nx-claude-plugins` from `nrwl/nx-ai-agents-config`, for the Nx MCP
  server and `nx-generate` skills), and a block in CLAUDE.md. Kept the
  allowances and ignores; Siggie dropped the plugin, and I removed the CLAUDE.md
  block: it said to ALWAYS scaffold with `nx-generate`, which contradicts the
  §Nx rule against generators adding tsconfig `paths` / `project.json` (the
  rule exists for app extractability).
- **pnpm in the sandbox.** Siggie chose to grant it rather than keep running
  installs. Siggie asked whether that would allow npm publishing: yes as first
  proposed — `~/.npmrc` holds a token and was readable, and a domain allowlist
  doesn't distinguish methods. Fix: `~/.npmrc` under
  `sandbox.credentials.files` with `mode: deny` (the mechanism Siggie's global
  settings already use for `~/.ssh`; my first guess, `filesystem.denyRead`,
  was not their established key), plus `permissions.deny` for publish
  commands as a weak second layer.
- The first grant was `~/Library/pnpm/store` only. A frozen no-change install
  then chose an in-repo `.pnpm-store` and tried to delete `node_modules` to
  reinstall (aborted: no TTY; nothing lost; the empty stray store removed).
  pnpm tests writability of the store's PARENT. Widened to `~/Library/pnpm`
  (which holds only `store/`); `pnpm store path` and a frozen install then
  used `~/Library/pnpm/store/v10`. Pinning `store-dir` in the repo was
  rejected — it would commit a machine path.

## 2026-09-30 — design of the package split (PLAN §3.1)

- Read all of `apps/dmvd/src/help/`, its call sites and the tests, then wrote
  the split's design into [docs/PLAN.md](docs/PLAN.md) §3.1 rather than
  starting code: Siggie asked whether to plan first and implement in a fresh
  session, since this one was long and mostly setup.
- Found the plan wrong about the tour's host callbacks: it named
  `onApplyState`/`onReadState` (the absolute-state model dmvd removed on
  2026-08-27); the code has `onPushChange`/`onPopChange`/`onJumpChanges`/
  `onTourStart`/`onTourEnd`. Fixed in §3's table and §5.
- Judgement calls in §3.1 that were NOT obvious, with the reason:
  - `<Prose>` not `<HelpMarkdown>`/`<Markdown>`: the package isn't "help", and
    `Markdown` clashes with react-markdown's default import at call sites.
  - Cut entry lines at `Beats:` in the tour parser, instead of giving generic
    field readers a stop-at option: keeps markdown and anchor ignorant of
    beats with identical output (beat fields are indented; `extractField`
    today stops at a `beats` field by name, parked or not).
  - Help-mode remainder reads `content` from the tour context rather than
    reparsing: temporary, dmvd-internal, avoids a second fill pass.
  - `Once:` in anchor, not markdown or tour: width must be computed from the
    stripped text, and any popover can carry one.
  - Keep `help-*` class names through the split: renaming touches dmvd's
    theme, e2e and tests, and would muddy "each stage changed nothing".
- `CLAUDE.md`'s symlink-trap note said to expect it on the first vitest run;
  it did not happen, and the note now says so.

- Siggie accepted the markdown-owns-the-document-structure answer (not the
  one-parser-per-package alternative). PLAN §1 lists it as decided; the §2
  box and the "provisional" wording in §3.1 were rewritten to state it.
