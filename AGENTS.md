# AGENTS.md

Guidance for coding agents working in this repo.

## What this repo is

`@fission-ai/openspec` — a Node CLI (Commander) that manages spec-driven
development: `openspec/` changes and specs, plus generated slash commands and
skills for 30+ AI tools. The repo dogfoods itself: it manages its own work with
its own `openspec/` tree.

## Stack and build

- pnpm 10 (`packageManager`), Node >= 20.19. Pure ESM, `"module": "NodeNext"`.
  **Every relative import must carry a `.js` extension** (`../core/config.js`),
  including ones that point at `.ts` files.
- `pnpm build` runs `build.js`, which wipes `dist/` and runs the local `tsc`.
  There is no bundler; `dist/` mirrors `src/`.
- `pnpm dev:cli` = build then run `bin/openspec.js`. Use it to exercise the real
  CLI against a scratch directory.
- `strict` TypeScript. `tsconfig.json` **excludes `test/`**, so
  `pnpm exec tsc --noEmit` does not typecheck tests, and vitest transpiles
  without typechecking. Type errors in test files surface only at runtime.

## Verify like CI does

```bash
pnpm build            # tests run against dist/, so build first
pnpm test
pnpm exec tsc --noEmit
pnpm lint             # eslint src/ only
```

Those four are exactly CI (`linux-bash`, `macos-bash`, `windows-pwsh`).
Lint and typecheck cover `src/` only — nothing lints or typechecks `website/`.

Focused runs:

```bash
pnpm exec vitest run test/path/to/file.test.ts
pnpm exec vitest run test/path/to/file.test.ts -t "case name"
VITEST_MAX_WORKERS=1 pnpm test     # vitest.config.ts caps workers at 4 otherwise
```

## Testing gotchas

- **Stale `dist/` silently runs old code.** `vitest.setup.ts` only builds when
  `dist/cli/index.js` is *missing*, never when it is out of date. Anything that
  spawns the CLI (`test/helpers/run-cli.ts`, all of `test/cli-e2e/`) exercises
  the build, not your edits. Rebuild before trusting a result.
- `test/AGENTS.md` is scoped, load-bearing guidance for writing tests here:
  cross-platform path expectations and path canonicalization. Read it before
  adding or changing a test.
- vitest forces `pool: 'forks'` (tests assume per-file process isolation), and
  sets `OPENSPEC_TELEMETRY=0` / `DO_NOT_TRACK=1` so spawned CLI runs don't
  touch your real global config or phone home. Keep that property when adding
  helpers.
- Windows is a first-class CI target. Never hard-code `/` in paths or expected
  output; use `path.join` / `FileSystemUtils.toPosixPath()`, and canonicalize
  with `fs.realpathSync.native` (tests) or
  `FileSystemUtils.canonicalizeExistingPath()` (product code) before comparing
  path identity.

## Generated files and parity traps

Several committed trees are generated. Editing them by hand is wasted work;
editing their source without regenerating fails CI.

| Committed artifact | Regenerate with | Guard |
|---|---|---|
| `skills/*/SKILL.md` (skills.sh distribution) | `pnpm build && pnpm generate:skills` | `test/core/templates/skillssh-parity.test.ts` |
| Golden hashes in `test/core/templates/skill-templates-parity.test.ts` | `pnpm build && pnpm regen:parity-hashes` | same test (the test is the authority; always run it after) |
| `openspec/specs/*/spec.md` (written by `openspec archive`) | — | `test/specs/source-specs-normalization.test.ts` |

- All slash-command and skill markdown originates in
  `src/core/templates/workflows/*.ts` + `src/core/templates/skill-templates.ts`.
  Change the template, not the output.
- `docs-lab/reference/schemas/spec-driven/index.md` quotes `schemas/spec-driven/schema.yaml`
  instruction blocks **verbatim**; `test/core/templates/schema-docs-instruction-parity.test.ts`
  fails on any drift. Same for `test/apply-docs-claims.test.ts`, which asserts
  specific claims in `docs-lab/reference/cli.md` and `skills.md`.
- `skills/**` is pinned to LF in `.gitattributes`; don't normalize line endings.
- `.opencode/` and `opencode.json` are gitignored — they're `openspec init` /
  `openspec update` output for this repo. `.agents/skills/` *is* committed and
  hand-authored.

## Docs layout

- `docs-lab/**` is the authored source for the documentation site.
  `website/docs.sync.config.mjs` decides which pages are published, their slugs,
  and sidebar order — a page missing from it is invisible even if written.
  Held-back sections are commented out in that config, not deleted.
- `website/` is a separate pnpm project with its own lockfile and is **not** a
  member of the root workspace (`pnpm-workspace.yaml` lists only `.`). CI audits
  and validates its lockfile separately.
- `docs/**` is the legacy tree; `docs-lab/sources.md` holds the old→new mapping.
  New docs go in `docs-lab`.
- Repo-local skills for docs work: `.agents/skills/write-openspec-docs`,
  `verify-openspec-docs`, `draft-openspec-docs`, `release-openspec`.

## Adding an AI tool

Compare real tool-add commits (`git show 781c7f9` warp, `d1642cb` easycode,
`c879d13` code studio) rather than guessing the file list. The consistent ones:

- `src/core/config.ts` — `AI_TOOLS` (and `OPENSPEC_SKILL_NAMES` only if the
  tool adds skill names). Every commit touches this.
- Command-generating tools only: `adapters/<tool>.ts` + `registry.ts` +
  `adapters/index.ts`. **Skills-only tools (warp, grok build, GSD) need none of
  these** — check whether the tool gets commands before creating an adapter.
- `src/core/command-surface.ts` — what the tool's command surface can hold.
- `src/utils/command-references.ts` — invocation spelling.
- `docs-lab/reference/supported-tools.md`, tests, and a changeset.

A change proposal is expected here: these commits ship
`openspec/changes/<name>/{proposal,design,tasks,specs/}` *and* update
`openspec/specs/ai-tool-paths/spec.md` (and `cli-init`, `command-generation`)
in the same PR — see `e232080`. Proposal-first is not aspirational; feature
commits in this repo carry them.

## Dependencies and Nix

- Security overrides live **only** in `pnpm-workspace.yaml`. A `pnpm.overrides`
  block in `package.json` does not merge with it — pnpm 10 uses one *instead of*
  the other.
- Any change to `pnpm-lock.yaml` (including `pnpm update`) requires
  `bash scripts/update-flake.sh` and committing the updated `flake.nix`, or the
  `nix-flake-validate` CI job fails. Dependabot cannot produce that hash.
- Committed GitHub Actions are SHA-pinned. Keep that convention.

## Workflow expectations

From `CONTRIBUTING.md` (the authority; read it before opening a PR):

- Bug fixes, typos, and small improvements go straight to a PR.
- New features, significant refactors, and architecture changes need an
  OpenSpec change proposal first — a PR containing only
  `openspec/changes/<name>/`, approved before code is written.
- Every change starts with an issue or a discussion; link it from the PR.
- Run `pnpm changeset` for anything user-facing and commit the file. CI
  validates it with `changeset status --since=origin/main`.
- Branch off `main`; conventional-commit PR titles (`fix(archive): ...`).
- Release is changesets-driven via a Version Packages PR; you don't tag.

## ESLint rules that encode real bugs

`src/**` may not statically import `@inquirer/*` — those modules keep the event
loop alive when stdin is piped and hang pre-commit hooks (#367). Use dynamic
`import()`. `src/core/init.ts` is the single documented exception (it's
dynamically imported from `src/cli/index.ts`, so it never loads at CLI startup).
