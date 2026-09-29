# Wiki: the job data pipeline

A map for finding code quickly. It covers `packages/{crawler,sources-ats,sources-other,boards,parsers,static-data,store}`.
Contract and rule text lives in `docs/INTERFACES.md`, and this page does not repeat it. Links: §1.2 import graph, §1.4
additive-only rules, §1.5 true-or-unknown, §3 table owners, §8 package exports, §9 outbound hosts, §10 never-crawl hosts.
Outcomes: `docs/outcomes/<pkg>.md`. Source notes: `docs/sources/README.md`. All paths below are relative to the repo root. Checked 2026-09-29.

## 0. What actually runs today (read first)

The owning packages are Built, but the app server (`apps/server`) wires only part of them. The rest is covered by INTERIM stand-ins:

| Stage | Package design | What `apps/server` runs now |
|---|---|---|
| Board list / schedule | boards `BoardService` + `CrawlScheduler` (tables `board_prefs`, `crawl_runs`…) | `apps/server/src/interim/boards.ts` (tables `srv_boards`, `srv_crawl_runs`). It ticks every 60 s (`:323-344`) and crawls ALL boards once `settings.crawl.intervalHours` has passed |
| Link → board resolve | boards `BoardService.resolve` | the same (real), `apps/server/src/app.ts:148` `boardResolver()` |
| Crawl | crawler `crawl()` + sources-ats `allSources()` | real, in a worker thread: `apps/server/src/interim/crawl-worker.ts:64`, through sources-ats `politeFetch`. No `Runs` and no crawler `Scheduler` |
| Feeds (RemoteOK, HN…) | sources-other `SourceService` | **not wired** (the server never imports sources-other) |
| Add a job by link/text | sources-other `jobFromUrl` | `apps/server/src/interim/external.ts` (writes `ats='external'` rows through crawler `Store.upsertJob`, `:322`) |
| Search / filters | store `JobStore.search` (RAM typed arrays + 3 FTS tables) | `apps/server/src/interim/jobs.ts`: SQL over `srv_job_index` (trigger-fed copy of `jobs`) + crawler `jobs_fts` (`buildWhere` `:404`) |
| Store copy | store `syncFromCrawler()` | **not called** by the server (only `packages/store/src/cli.ts:220`) |

## 1. Flow

```
 packages/boards/data/board-directory.json      (3,581 rows; loadActiveDirectory  boards/src/directory.ts:148)
        |  seedBoards()  apps/server/src/core/seed.ts   /  user paste -> BoardService.resolve  boards/src/service.ts:435
        v                                                    (detectBoardFromUrl detect.ts:226 -> detectBoard discover.ts:148)
 board list: srv_boards (server interim)  |  board_prefs + board_checks (boards pkg)
        |  schedule: interim/boards.ts:323 tick  |  boards CrawlScheduler scheduler.ts:85  |  crawler planDue scheduler.ts:81
        v
 crawl(boards, {store, http, sources})                                   crawler/src/crawl.ts:172
   |- runPool: <=3 boards per host, <=8 in total                         crawl.ts:134
   |- skip: no adapter / bad origin / host tripped / cooldown / budget   crawl.ts:273-311
   |- HttpClient (the only network code)                                 crawler/src/http.ts:356
   |     forbiddenHostOf (never + held_back) hosts.ts:37, private-address check, robots.txt robots.ts:23 (24 h cache),
   |     Pacer >=1 s + 100 ms per host http.ts:211, no redirects followed, ETag/If-Modified-Since (conditional sources)
   |     [server: fetchImpl = sources-ats politeFetch polite-fetch.ts:139, adds Retry-After, Crawl-delay cap, Latin-1]
   v
 Source.fetchBoard(board, http) -> RawJob[]      types.ts:37 RawJob, :78 Source
   crawler built-ins sources/{greenhouse,lever,ashby}.ts  -> wrapped by sources-ats adapters/builtins.ts
   sources-ats adapters/{workable,recruitee,personio,teamtailor,gem}.ts
   every sources-ats adapter -> polishSource/polishRaw (clean text, EU pay)  sources-ats/src/polish.ts:12,31
   v
 normalizeJob(board, raw) -> Job | null           crawler/src/job.ts:136
   skipReason (no id / title / http link) :55;  htmlToText, parseLevel, parsePayFromText (parsers);
   optional hooks tryHook('parsePlaces'|'parseYearsRequired'|'parseStatements') :22-35;  roleKeyOf :113; contentHash
   v
 Store.upsertJob(job, now)   ONE transaction per board          crawler/src/store.ts:387
   same (ats,board,job_id) -> update / unchanged (last_seen only);  same canonical_url+dedup_hash on ANOTHER board
   -> 'dupUrl' credit in job_sources;  same role_key on another board -> duplicate_of set
   v
 close-vanished:  proven = fullBoardListing && boardQualifies && ingested>0 && not region-ambiguous   crawl.ts:377
   recordReading -> applyMisses (2 readings, confirmGap 2 h | grace 48 h, mass-close hold >50% of >=10)  store.ts:491,509
   empty board 3x over 7 d -> closeBoardEmpty 'board_empty';  404 3x over 14 d -> 'board_gone'          crawl.ts:391,447
   v
 SQLite  $JOBLEFT_HOME/data/jobleft.db:  jobs, jobs_fts (triggers store.ts:120), boards, job_sources
   v
 search:  server interim  srv_job_index + jobs_fts  (apps/server/src/interim/jobs.ts:311,404)
          store pkg       syncFromCrawler jobstore.ts:178 -> store_jobs/job_docs/job_*_fts -> search.ts (RAM facets)
```

## 2. Packages

### crawler (`packages/crawler/src`), which owns `jobs, jobs_fts, boards, job_sources, crawler_runs, crawler_run_boards, crawler_hosts, crawler_robots, crawler_meta`
Fetches, normalises, stores and closes postings. Imports contracts and parsers. How to run it: `packages/crawler/README.md`.
- `crawl.ts:172` `crawl()`: the orchestrator. `:27-30` defaults (confirm 2 h, not-found 14 d, 10k jobs/board).
- `job.ts:136` `normalizeJob`, `:42` `NORMALIZER_VERSION` (=5), `:113` `roleKeyOf`.
- `store.ts:285` `Store`: `:334` migrate (v1 `:60`, v2 columns `:137-149`, v2 tables `:150`), `:387` upsertJob, `:491` recordReading, `:509` applyMisses, `:593` recordFailure, `:281` `ATS_NAMES`.
- `http.ts:356` `HttpClient`, `:211` `Pacer`, `:39` `USER_AGENT`, `:41` `PRODUCT_TOKEN`; `transport.ts:60` `nodeTransport`; `robots.ts:23`.
- `hosts.ts:12` NEVER list, `:20` `HELD_BACK_FAMILIES`, `:37` `forbiddenHostOf`.
- `lifecycle.ts:62` `boardQualifies`, `:101` `emptyFeedShouldClose`, `:111` `closeTooBroad`.
- `scheduler.ts:81` `planDue`, `:53` `retryDelayMs`, `:159` `httpForRun`, `:179` `runOnce`, `:315` `Scheduler`, `:450` `simulate`.
- `config.ts:52` `DEFAULT_CONFIG` (refreshHours 24, confirmHours 2, graceHours 48…); `sources/index.ts:10` `SOURCES`, `:13` `hostFor`.
- `contract.ts:60` `toContractJob`, `:153` `queryJobs`, `:136` `ftsQuery`; `normalize.ts:45` `canonicalizeUrl`; `runs.ts:51` `Runs`.
- CLI `cli.ts:619` (run/status/jobs/daemon/simulate/verify/clock/report/search). Mock boards: `packages/crawler/testkit/mock-boards.ts`.

### sources-ats (`packages/sources-ats/src`), no tables
Adds the Workable/Recruitee/Personio/Teamtailor/Gem adapters, repairs the 3 crawler built-ins, ATS detection from a URL, and the source list. Imports contracts, parsers, crawler.
- `registry.ts:13` `ATS_SOURCES`, `:22` `allSources()` (**always use this**, not the crawler's `SOURCES`).
- `adapters/*.ts`: one `Source` each, e.g. `adapters/workable.ts:74`. Built-in wrappers: `adapters/builtins.ts:57,76,121`.
- `polish.ts:12` `polishRaw`, `:31` `polishSource`; `pay-text.ts` `parseEuropeanPay`.
- `detect.ts:72` `detectAts`, `:251` `neverContactHost`, `:278` `classifyUrl`; `hosts.ts:35` `atsHost`, `:54` `boardHost`, `:19` `SUBDOMAIN_FAMILIES`.
- `source-list.ts:23` `ATS_SOURCE_DETAILS` (a test checks it against `docs/sources/*.md`); `polite-fetch.ts:139` `politeFetch`, `:249` `BoundedPacer`.
- `report.ts` `buildHealthReport`, `plainReason`; CLI `cli.ts` (sources/detect/crawl/jobs/report/standin).

### sources-other (`packages/sources-other/src`), which owns `source_state, source_runs, source_requests, source_host_slots, feed_postings`
Non-ATS feeds (each OFF by default), add-a-job, and the metered fetch. Imports contracts, parsers, crawler. Not wired into the server (§0).
- `catalog.ts:21` `ALL_FEEDS`; `feeds/*.ts` one file per feed; `types.ts` `JobFeed`/`FeedResult`.
- `runner.ts:256` `refreshSources`, `:127` `applyFeedResult`: writes crawler `jobs` with `ats='feed:<id>'` via `normalizeJob` + `Store.upsertJob`, plus a `feed_postings` row per source.
- `service.ts:54` `SourceService`; `limits.ts` per-source run limits; `http.ts:171` `FeedClient`, `:21` `NEVER_CRAWL`, `:98` `DbPacer`.
- `view.ts:165` `feedJobs` (reference reader); `external.ts:83` `jobFromUrl`, `:148` `jobFromText`; `db.ts:125` migrate (tables `:20-71`).

### boards (`packages/boards/src`), which owns `board_prefs, board_checks, crawl_runs, crawl_board_reports, board_pending_links, host_pacing, robots_cache`
The board directory, the person's boards, link → board resolution, board health and back-off, and the scheduler. Imports contracts, crawler, sources-ats, static-data.
- `service.ts:124` `BoardService` (`:435` resolve, `:291` add, `:401` due, `:390` recordCheck), `:89` `backoffMs`.
- `scheduler.ts:85` `CrawlScheduler` (batches of 40, `:83` grace 24 h); `db.ts:17` MIGRATIONS, `:100` `migrateBoards`.
- `detect.ts:226` `detectBoardFromUrl` (pure), `:11` `PROVIDER_NAMES`, `:263` `boardApiUrl`; `page.ts:135` `scanPage`; `discover.ts:148` `detectBoard`.
- `directory.ts:148` `loadActiveDirectory`, `:196` `BoardDirectory`; `refresh.ts:101` `refreshDirectory`; `verify.ts:130` `verifyBoard`.
- `http.ts:205` `createBoardHttp`, `:54` `SqlitePacer` (1.1 s, shared across processes); `sources.ts:13` `unreadableRegion` (Greenhouse EU refused).
- Data: `packages/boards/data/board-directory.json`, `packages/boards/data/board-directory-pruned.json`; scripts `packages/boards/scripts/`.

### parsers (`packages/parsers/src`), pure functions, no tables
Reads the facts of a posting (pay, places, work model, level, years, statements), each with evidence or null. Imports contracts only.
- `facts.ts:84` `extractFacts`: the "one call", but **no production code calls it** (§5).
- `board.ts:310` `fromRawJob`, `:264` `postingsFromBoard`, `:61-206` `fromGreenhouse…fromJsonLd`.
- `pay.ts:603` `parsePay`, `:644` `parsePayFromText`, `:44` `annualize`; `pay-filter.ts:22` `payMeetsMinimum`.
- `places.ts:630` `parsePlaces`, `:347` `parseLocationText`; `workmodel.ts:199` `parseWorkModel`; `years.ts:83` `parseYearsRequired`.
- `level.ts:245` `parseLevel`, `:164` `levelFromTitle`; `statements.ts:31` `parseStatements`; `html.ts:56` `htmlToText`.

### static-data (`packages/static-data/src`), which owns `company_facts, company_fact_labels`
Shipped datasets (H-1B, places), `companyKey`, company facts, and signed dataset releases. Imports contracts.
- `company-key.ts:79` `companyKey`; `index.ts:64` `loadH1bIndex`, `:69` `loadPlaceIndex`, `:73-74` `loadSkills`/`loadDirectoryRows` (STUBS, they throw).
- `facts/company-facts.ts:80` `migrateCompanyFacts` (tables `:88`, `:101`), `:125` `CompanyFacts`; `routes.ts:45` `createStaticDataRoutes`.
- `h1b/tag.ts` `h1bTagFor`/`passesH1bFilter`; `datasets/release.ts` `installReleases`. Data: `dist/*.json.gz`, `data/*.json`.

### store (`packages/store/src`), which owns the store tables in the §4 table
Opens the DB (`openDatabase`, used by the server), keeps its own copy of every job, search, fit vectors, and the person's records. Imports contracts, crawler, static-data.
- `db.ts:30` `openDatabase` (page_size 16384, WAL), `:76` MIGRATIONS, `:224` `migrate`.
- `jobstore.ts:39` `JobStore` (`:63` search, `:85` upsertJobs, `:102` refreshScope, `:178` syncFromCrawler, `:205` crawlRowToInput).
- `record.ts:140` `normalizeInput`, `:382` `facetsOf`, `:438` `packFacets` (fixed byte layout); `memindex.ts:108` `setRow` (reads the same offsets).
- `search.ts:128` `compileFilter`, `:231` `passes`; `writer.ts:75` `JobWriter`; `fit.ts` `FitIndex`; `service.ts:29` `StoreService`.
- `index.ts` `makeJobId` → `"<ats>:<board>:<externalId>"` (ats and board lower case).

## 3. How rows get written (identity and dedupe)

| Key | Where | Meaning |
|---|---|---|
| `UNIQUE(ats, board, job_id)` | `crawler/src/store.ts:96` | identity. Two ids on one board are always two jobs |
| `canonical_url` + `dedup_hash`, another board | `store.ts:425` | the same posting reached another way: credit it in `job_sources`, no new row |
| `role_key` (company+title+places+text+pay hash) | `store.ts:379` | a repeat of the role on another board: row stored, `duplicate_of` set; released when the canonical row closes (`:455`) |
| `content_hash` | `job.ts:258` | an unchanged posting only gets `last_seen` updated |
| validators `url#normalizer=N` | `crawl.ts:320` | a `NORMALIZER_VERSION` bump forces a full re-read instead of a 304 |

## 4. DB table cheat-sheet (from code)

| Table | Owner | Purpose | Defined in |
|---|---|---|---|
| `schema_migrations` | shared (each owner's rows) | `(owner, version)` steps | every migrate fn |
| `jobs` | crawler | every posting (ATS, `feed:<id>`, `external`); soft close `closed_at` | `crawler/src/store.ts:61,137` |
| `jobs_fts` | crawler | FTS5 external-content on title/company/location/description | `store.ts:120` |
| `boards` | crawler | per-board health, validators, hold/not-found streaks. PK `(ats,board)` (no region!) | `store.ts:103,144` |
| `job_sources` | crawler | who listed a posting (credits) | `store.ts:157` |
| `crawler_runs`, `crawler_run_boards` | crawler | resumable runs (not used by server) | `store.ts:170,191` |
| `crawler_hosts`, `crawler_robots`, `crawler_meta` | crawler | host waits/last request, robots cache, lease + clock offset | `store.ts:217,223,233` |
| `source_state`, `source_runs`, `source_requests`, `source_host_slots`, `feed_postings` | sources-other | feed on/off, limits, pacer, per-source posting facts | `sources-other/src/db.ts:20-71` |
| `board_prefs`, `board_checks`, `crawl_runs`, `crawl_board_reports`, `board_pending_links`, `host_pacing`, `robots_cache` | boards | the person's boards, check health, refresh history, offline pastes, shared pacer, robots 10 min | `boards/src/db.ts:20-88` |
| `company_facts`, `company_fact_labels` | static-data | company facts with source and date, Wikidata labels | `static-data/src/facts/company-facts.ts:88,101` |
| `store_meta`, `store_jobs`, `job_docs`, `job_keys`, `job_tombstones`, `facet_tags`, `companies` | store | the store's job copy: packed facets blob, compressed doc, dedupe keys | `store/src/db.ts:80-120` |
| `job_head_fts`, `job_body_fts`, `job_title_fts` | store | contentless FTS5 | `store/src/db.ts:134-139` |
| `job_vectors`, `fit_runs` | store | float16 bge-small vectors + embed hash | `store/src/db.ts:143,153` |
| `tracker*`, `saved_filters`, `profile`, `settings`, `chats`, `chat_messages`, `notifications` | store | the person's records (server uses `srv_*` stand-ins instead) | `store/src/db.ts:161-197` |
| `srv_job_index` (+3 triggers on `jobs`) | server INTERIM | narrow copy of open non-duplicate jobs for search | `apps/server/src/interim/jobs.ts:311-345` |

## 5. How do I…

**Add a new ATS source adapter** (for example `jobvite`)
1. Write `docs/sources/<ats>.md` and add a row to `docs/sources/README.md` (evidence, robots, terms) first. Owner approval if the family is held back.
2. `packages/contracts/src/job.ts:13` add the id to `CRAWL_ATS_IDS` (and `ATS_IDS` `:19`). Then run `pnpm --filter @jobleft/contracts run gen`.
3. `packages/sources-ats/src/adapters/<ats>.ts`: `map<Ats>()` → `RawJob`, and `export const x: Source = polishSource({ ats, fullBoardListing, host: boardHost, fetchBoard })`. Copy `adapters/workable.ts`. Throw `FeedFormatError` when the list field is missing, and return `unreadableJob()` for rows with no id or title.
4. Register it in `sources-ats/src/registry.ts:13` and export it from `index.ts`. Hosts: `hosts.ts:35` `atsHost` (+ `SUBDOMAIN_FAMILIES` if the board is a sub-domain). URL detection: `detect.ts:72`. Source list: `source-list.ts:23`.
5. boards: `detect.ts:11` `PROVIDER_NAMES` (typed `Record<CrawlAtsId,…>`, so tsc flags it), link shapes in `detectBoardFromUrl` `:226`, `boardApiUrl`/`boardApiHost` `:263-279`.
6. crawler: `store.ts:281` `ATS_NAMES` (the credit name. teamtailor/gem are already missing). Optional: a `fromX` reader in `parsers/src/board.ts`.
7. Add the host to `docs/INTERFACES.md` §9. Add a fixture under `packages/sources-ats/test/fixtures/standin/<ats>/` plus a test. Then `pnpm --filter @jobleft/sources-ats test`.
   The server picks it up on its own through `allSources()` (`apps/server/src/interim/crawl-worker.ts:31`).

**Add a new parsed field to jobs** (end to end)
1. Parser: a pure function in `packages/parsers/src/<x>.ts` returning `value | null` + `FactEvidence`, exported from `index.ts`. Wire it into `extractFacts` (`facts.ts:84`) too, for consistency.
2. Contract: an optional/nullable field on `JobSchema` (`packages/contracts/src/job.ts:165`), and in `JobEvidenceSchema` `:100` if it has evidence. Run `pnpm --filter @jobleft/contracts run gen` and `node scripts/gen-interfaces.ts`.
3. Crawler: add the field to `Job` in `crawler/src/types.ts:99`. Compute it in `normalizeJob` (`job.ts:136`), directly or through `tryHook`. Add it to the `contentHash` list `:258`. **Bump `NORMALIZER_VERSION`** `:42`.
4. Crawler schema: a new step `v < 3` in `store.ts:334` (ALTER TABLE ADD COLUMN only). Bump `CRAWLER_SCHEMA_VERSION` `:58` (the server's `KNOWN_OWNERS` imports it, `apps/server/src/db/schema.ts:223`). Add the column to both the UPDATE and the INSERT in `upsertJob` `:400,434`.
5. Read path (server, live): `apps/server/src/interim/jobs.ts:167` `rowToJob`; `crawler/src/contract.ts:60` `toContractJob` (package readers).
6. Store package (for later): `store/src/jobstore.ts:205` `crawlRowToInput`, `record.ts:140` `normalizeInput`.
7. sources-other feeds: a `feed_postings` column (new `db.ts` step) if the feed states the fact itself.

**Change crawl cadence or politeness**
- Live app cadence: `settings.crawl.intervalHours`, default **6** at `apps/server/src/services/kv.ts:30` (range 1-168 in `contracts/src/api.ts:143`). The tick is `apps/server/src/interim/boards.ts:323-344`.
- Crawler CLI/daemon: `crawler/src/config.ts:52` `DEFAULT_CONFIG` (refresh 24 h, confirm 2 h, grace 48 h, the per-host limits), the retry ladder `scheduler.ts:51-58`, the planner `planDue` `:81`.
- boards scheduler: `boards/src/scheduler.ts:83` grace 24 h, batch 40. Board back-off `service.ts:89` (1 d doubling to 30 d).
- Per-host gap: `Pacer` `crawler/src/http.ts:211` (1 s + 100 ms margin); server worker `apps/server/src/interim/crawl-worker.ts:24-62` (`BoundedPacer` caps Crawl-delay at 60 s, timeout 20 s, 5000 requests/run); `politeFetch` `sources-ats/src/polite-fetch.ts:139`. boards `SqlitePacer` 1100 ms (`apps/server/src/app.ts:155`).
- Closing thresholds: `crawl.ts:27-30`, `lifecycle.ts:11-17,100,110`, mass-close in `store.ts:518-533`. Never loosen these without reading crawler O3/O4.
- Never-crawl: `crawler/src/hosts.ts:12-26`. The same list is copied in `sources-ats/src/detect.ts:251`, `sources-other/src/http.ts:21` and `boards/src/hosts.ts`. Change all four together (and INTERFACES §10).

**Add a new filter facet**
1. Contract: an optional field on `JobFilterSchema` (`packages/contracts/src/filter.ts:27`). If unknowns matter, add a value to the `includeUnknown` enum `:59`.
2. Live server: add the column to `srv_job_index` (`apps/server/src/interim/jobs.ts:316-343`), and the clause in `buildWhere` `:404` (pattern: `minAnnualPayUsd` `:452`). **`ensureIndexes` returns early at `:321` once the table exists**, so a new column needs a new existence check (or a version key) to force the rebuild.
3. UI: `apps/ui/src/lib/filters.ts`.
4. Store package (for later): `record.ts:360` `Facets`/`facetsOf` `:382`/`packFacets` `:438` and `memindex.ts:108` `setRow` share byte offsets, so change both. `search.ts:78` `Compiled`, `compileFilter` `:128`, `passes` `:231`.

## 6. Gotchas

- **Two search stacks.** Production search is `apps/server/src/interim/jobs.ts` (SQL + `jobs_fts`). The store package's RAM search, fit order and dedupe are NOT in the request path. `syncFromCrawler` is never called by the server.
- **`syncFromCrawler` drops schema-2 facts.** `crawlRowToInput` (`store/src/jobstore.ts:183-187,205`) selects only v1 columns, so places, levels, years, statements, evidence, remote scope and `work_model` are lost in the store copy. Fix this before wiring the store in.
- **`extractFacts` is unused.** The crawler reads facts through `tryHook` name lookups (`crawler/src/job.ts:22-35`). A renamed parsers export, or a result that fails its schema check, silently turns the fact into "unknown" with no error. The crawler also keeps its own place parser (`crawler/src/places.ts`), used whenever the hook result is incomplete.
- **Server refuses a DB that has sources-other migrations.** `KNOWN_OWNERS` (`apps/server/src/db/schema.ts:223`) has no `'sources-other'`, and `db/open.ts:42` refuses any unknown owner. Running `SourceService` or the sources-other CLI on the app DB bricks startup until that entry is added.
- **User-Agent differs from the docs.** Code: `jobleft/0.1.3 (+https://github.com/Blueturboguy07/jobleft; no personal data)` (`crawler/src/http.ts:39`, `config.ts:10`). The robots token is still `jobleft-build` (`http.ts:41`). INTERFACES §8 and §11 still say `jobleft-build/0.1 (research build…)`.
- **Three default cadences:** server 6 h (`apps/server/src/services/kv.ts:30`), store `DEFAULT_SETTINGS` 12 h (`store/src/userdata.ts:322`), crawler 24 h (`config.ts:52`). Grace also differs: `crawl()` default 48 h, boards scheduler 24 h.
- **Closing needs proof.** Only a `fullBoardListing` source with a clean reading (no failure, ≥1 ingested, ≤5% unreadable) plus a second confirming reading closes anything. Failures, empty answers, 304s during a hold, and paged adapters close nothing. A board name listed under two regions in one run never closes (`crawl.ts:197-199,376`).
- **`boards` PK ignores region** (`store.ts:116`), but `crawl()` dedupes by region (`crawl.ts:170`). The same token in US and EU shares one health row.
- **Soft close only.** Rows are never deleted by the crawler; a reappearing posting reopens (`upsertJob` clears `closed_at`). The store package *purges* untracked closed jobs, the crawler does not.
- **Additive-only schema** (INTERFACES §1.4): add a column in a new numbered step. Never change a type or drop anything. Crawler v2 was written defensively (`if (!jc.has(name))`), so follow that pattern.
- **True-or-unknown** (§1.5): `normalizeJob` never defaults. An occupation-only level ("Cashier") is set to null on purpose (`job.ts:168`). Pay from text skips bonus/401k lines (`job.ts:63-76`). Board pay is never rounded.
- **Greenhouse ids > 2^53** are kept as strings (`http.ts:34` `keepBigIntegers`), so never `Number()` a job id.
- **`ATS_NAMES` lacks teamtailor/gem** (`store.ts:281`), so their credit reads "X careers (teamtailor)".
- **Greenhouse EU** boards are recognised but refused (`boards/src/sources.ts:13`). The EU host is not in §9 either.
- **Feeds reuse the crawler dedupe spine.** `applyFeedResult` calls `normalizeJob` with `ats='feed:<id>'`, board `<id>`. `feed_postings` facts override the text-derived columns only in `view.ts` readers.
- **Contracts gen**: after touching `packages/contracts/src`, run `pnpm --filter @jobleft/contracts run gen` and `node scripts/gen-interfaces.ts`. The GENERATED blocks in INTERFACES are copied from code.
