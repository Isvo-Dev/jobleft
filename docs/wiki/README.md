# jobleft wiki (navigation layer)

Added by the fork owner on 2026-09-29. The author's own docs stay the source of truth; these pages are a **short map
into them and into the code**, written so a new session can start work without re-reading ~90k lines.

## Read in this order

| Need | Read |
|---|---|
| What the app is, what's built | `README.md`, `docs/BUILD-REPORT.md` §1-3 |
| Server, UI screens, extension, shell, Windows dev loop | [apps.md](apps.md) |
| Crawler → parsers → store → search, DB tables | [pipeline.md](pipeline.md) |
| Contracts, match score, resume + truth gate, AI engine, assistant, network | [intelligence.md](intelligence.md) |
| CareerOS inventory (features, visuals, data model) | [careeros.md](careeros.md) (local only, not pushed) |
| How CareerOS maps onto jobleft, phases, open decisions | [merge-map.md](merge-map.md) (local only, not pushed) |
| Exact API routes, env vars, table owners, rules | `docs/INTERFACES.md` (large: grep a heading, don't read it whole) |
| Per-package acceptance criteria | `docs/outcomes/<package>.md` |
| Why things were built this way (Jobright parity plan) | `docs/PLAN.md` |

## Code graph

`graphify-out/GRAPH_REPORT.md`: 5,812 nodes, 173 communities, AST-only over `packages/` + `apps/` (tests, fixtures,
mocks and `apps/server/jobsync/` excluded). `graph.json` is git-ignored; rebuild it for `graphify query` with
`graphify update .` (free, no LLM, about 1 min). Most-connected types: `Job`, `Profile`, `Store`, `useApi()`, `AiEngine`.

## Facts checked on this Windows machine (2026-09-29)

- `pnpm install` OK (pnpm 12.4.1 via corepack, Node 24.13.0).
- Tests: 17 suites, 2 failures, both environmental. Node 24.13 prints `ExperimentalWarning: SQLite…`, which trips
  `packages/resume` "CLI walk-through" (stderr must not match `/at /`) and `apps/server` "--help, --version". Fix: Node ≥ 24.18.
- `pnpm start` with `JOBLEFT_OFFLINE=1 JOBLEFT_AUTO_CRAWL=0 JOBLEFT_SEED_BOARDS=none` builds the UI and serves on
  `127.0.0.1:47821`. Health is open; data routes answer 401 without a token and 403 from a foreign Origin.
- Stop: `POST /api/v1/shutdown` with header `x-jobleft-token` **and** `content-type: application/json`, body `{}` (415 otherwise).
- Launch token is in `.jobleft-dev/run/server.json`. Never paste it into chat, URLs you share, or commits.

## Top gotchas (details in the linked pages)

1. `apps/server/jobsync/` (~885 files) is a vendored reference fork of jobsync (Next.js/Prisma). Nothing runs it. Ignore it unless mining code.
2. The server runs **interim** stand-ins (`apps/server/src/interim/`, `srv_*` tables), not the store/boards package services the docs describe. (pipeline.md)
3. New API data = `LOCAL_API` in `packages/contracts/src/api.ts` + handler in `apps/server/src/routes.ts` `HANDLERS` + migration. tsc fails until all exist. (apps.md)
4. AI keys are memory-only on Windows (`apps/server/src/services/secrets.ts:98`). (merge-map Phase 0)
5. A fresh data folder + first profile save starts a **live crawl** of up to 120 boards. Use the env vars above for UI work.
6. UI must use hash routes and pass the CSP (no inline scripts, no remote fonts/APIs). (apps.md)
7. Contracts are additive-only; unknown keys pass validation silently. (intelligence.md)
