# jobleft (fork: Isvo-Dev/jobleft, upstream: blueturboguy07/jobleft)

Local-first job-search desktop app (pnpm monorepo, ESM TypeScript run directly by Node 24, Vite + React + Ant Design UI,
`node:http` + `node:sqlite` server, Tauri shell). The owner is merging CareerOS features into it (Windows 11, UK).

**Start here: `docs/wiki/README.md`.** It maps the author's docs and the code; read the wiki page for your area before
opening source. `docs/INTERFACES.md` is 1,575 lines, so grep a heading, don't read it whole.

## Commands (repo root)

- `pnpm install` · `pnpm typecheck` · `pnpm test` · one package: `pnpm --filter @jobleft/<name> test`
- Run: `$env:JOBLEFT_OFFLINE='1'; $env:JOBLEFT_AUTO_CRAWL='0'; $env:JOBLEFT_SEED_BOARDS='none'; pnpm start` (drop the env vars for real crawling)
- UI change: `pnpm --filter @jobleft/ui build`, reload the tab. Data: `.jobleft-dev/` (git-ignored).
- Code graph: `graphify-out/GRAPH_REPORT.md`; `graphify update .` rebuilds `graph.json` for `graphify query`.

## Rules

- Keep upstream merges cheap: new features in new files/packages; shared registries (`LOCAL_API`, `HANDLERS`,
  `KNOWN_OWNERS`, nav) get appended entries only. Contracts and migrations are additive-only.
- Fixes to upstream code and CareerOS ports go in separate commits.
- Never print or commit the launch token (`.jobleft-dev/run/server.json`) or `publik-app-token*`.
- Known env test failures on Node 24.13 (SQLite warning): resume "CLI walk-through", server "--help, --version". Not regressions.
- Update `docs/wiki/*` when you learn something structural a future session would otherwise rediscover.
